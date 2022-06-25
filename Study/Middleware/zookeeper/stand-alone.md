## @##Server

### @Main Class

Main class :   QuorumPeerMain

### Data Structure

```java
class ZKDatabase{
    protected DataTree dataTree;
    protected ConcurrentHashMap<Long, Integer> sessionsWithTimeouts;
}

class DataTree{
    private final NodeHashMap nodes;

    private IWatchManager dataWatches;

    private IWatchManager childWatches;
    
    /**
     * This hashtable lists the paths of the ephemeral nodes of a session.
     */
    private final Map<Long, HashSet<String>> ephemerals = new ConcurrentHashMap<Long, HashSet<String>>();

    /**
     * This set contains the paths of all container nodes
     */
    private final Set<String> containers = Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>());

    /**
     * This set contains the paths of all ttl nodes
     */
    private final Set<String> ttls = Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>());
}

public class NodeHashMapImpl implements NodeHashMap {
    private final ConcurrentHashMap<String, DataNode> nodes;
    private final boolean digestEnabled;
    private final DigestCalculator digestCalculator;
}

public class DataNode implements Record {
    // the digest value of this node, calculated from path, data and stat
    private volatile long digest;

    // indicate if the digest of this node is up to date or not, used to
    // optimize the performance.
    volatile boolean digestCached;

    /** the data for this datanode */
    byte[] data;

    /**
     * the acl map long for this datanode. the datatree has the map
     */
    Long acl;

    /** 
     * the stat for this node that is persisted to disk.
     */
    public StatPersisted stat;

    /**
     * the list of children for this node. note that the list of children string
     * does not contain the parent path -- just the last part of the path. This
     * should be synchronized on except deserializing (for speed up issues).
     */
    private Set<String> children = null;
}
```

#### Processors

1. PrepRequestProcessor
2. SyncRequestProcessor
3. FinalRequestProcessor

### Storage

* command       log

* data tree        snap

### Process

1. QuorumPeerConfig   ServerConfig初始化
2. 根据快照构造DataTree

#### handleIO

```java
// SlectorThread.java   
public void run() {
    try {
        while (!stopped) {
            try {
                // 处理事件
                select();
                processAcceptedConnections();
                processInterestOpsUpdateRequests();
            } catch (RuntimeException e) {
                LOG.warn("Ignoring unexpected runtime exception", e);
            } catch (Exception e) {
                LOG.warn("Ignoring unexpected exception", e);
            }
        }

        // Close connections still pending on the selector. Any others
        // with in-flight work, let drain out of the work queue.
        for (SelectionKey key : selector.keys()) {
            NIOServerCnxn cnxn = (NIOServerCnxn) key.attachment();
            if (cnxn.isSelectable()) {
                cnxn.close(ServerCnxn.DisconnectReason.SERVER_SHUTDOWN);
            }
            cleanupSelectionKey(key);
        }
        SocketChannel accepted;
        while ((accepted = acceptedQueue.poll()) != null) {
            fastCloseSock(accepted);
        }
        updateQueue.clear();
    } finally {
        closeSelector();
        // This will wake up the accept thread and the other selector
        // threads, and tell the worker thread pool to begin shutdown.
        NIOServerCnxnFactory.this.stop();
        LOG.info("selector thread exitted run method");
    }
}

private void select() {
    try {
        selector.select();

        Set<SelectionKey> selected = selector.selectedKeys();
        ArrayList<SelectionKey> selectedList = new ArrayList<SelectionKey>(selected);
        Collections.shuffle(selectedList);
        Iterator<SelectionKey> selectedKeys = selectedList.iterator();
        while (!stopped && selectedKeys.hasNext()) {
            SelectionKey key = selectedKeys.next();
            selected.remove(key);

            if (!key.isValid()) {
                cleanupSelectionKey(key);
                continue;
            }
            if (key.isReadable() || key.isWritable()) {
                // 处理读写事件
                handleIO(key);
            } else {
                LOG.warn("Unexpected ops in select {}", key.readyOps());
            }
        }
    } catch (IOException e) {
        LOG.warn("Ignoring IOException while selecting", e);
    }
}

 private void handleIO(SelectionKey key) {
     IOWorkRequest workRequest = new IOWorkRequest(this, key);
     NIOServerCnxn cnxn = (NIOServerCnxn) key.attachment();

     // Stop selecting this key while processing on its
     // connection
     cnxn.disableSelectable();
     key.interestOps(0);
     touchCnxn(cnxn);
     // WorkService workerPool
     workerPool.schedule(workRequest);
 }

// WorkService.java
public void schedule(WorkRequest workRequest) {
    schedule(workRequest, 0);
}

public void schedule(WorkRequest workRequest, long id) {
    if (stopped) {
        workRequest.cleanup();
        return;
    }

    ScheduledWorkRequest scheduledWorkRequest = new ScheduledWorkRequest(workRequest);

    // If we have a worker thread pool, use that; otherwise, do the work
    // directly.
    int size = workers.size();
    if (size > 0) {
        try {
            // make sure to map negative ids as well to [0, size-1]
            int workerNum = ((int) (id % size) + size) % size;
            ExecutorService worker = workers.get(workerNum);
            worker.execute(scheduledWorkRequest);
        } catch (RejectedExecutionException e) {
            LOG.warn("ExecutorService rejected execution", e);
            workRequest.cleanup();
        }
    } else {
        // When there is no worker thread pool, do the work directly
        // and wait for its completion
        scheduledWorkRequest.run();
    }
}

// ScheduledWorkRequest.java
private class ScheduledWorkRequest implements Runnable {

    private final WorkRequest workRequest;

    ScheduledWorkRequest(WorkRequest workRequest) {
        this.workRequest = workRequest;
    }

    @Override
    public void run() {
        try {
            // Check if stopped while request was on queue
            if (stopped) {
                workRequest.cleanup();
                return;
            }
            // IOWorkRequest workRequest
            workRequest.doWork();
        } catch (Exception e) {
            LOG.warn("Unexpected exception", e);
            workRequest.cleanup();
        }
    }

}
// IOWorkRequest.java
public void doWork() throws InterruptedException {
    if (!key.isValid()) {
        selectorThread.cleanupSelectionKey(key);
        return;
    }

    if (key.isReadable() || key.isWritable()) {
        cnxn.doIO(key);

        // Check if we shutdown or doIO() closed this connection
        if (stopped) {
            cnxn.close(ServerCnxn.DisconnectReason.SERVER_SHUTDOWN);
            return;
        }
        if (!key.isValid()) {
            selectorThread.cleanupSelectionKey(key);
            return;
        }
        touchCnxn(cnxn);
    }

    // Mark this connection as once again ready for selection
    cnxn.enableSelectable();
    // Push an update request on the queue to resume selecting
    // on the current set of interest ops, which may have changed
    // as a result of the I/O operations we just performed.
    if (!selectorThread.addInterestOpsUpdateRequest(key)) {
        cnxn.close(ServerCnxn.DisconnectReason.CONNECTION_MODE_CHANGED);
    }
}

// NIOServerCnxn.java
 void doIO(SelectionKey k) throws InterruptedException {
     try {
         if (!isSocketOpen()) {
             LOG.warn("trying to do i/o on a null socket for session: 0x{}", Long.toHexString(sessionId));

             return;
         }
         if (k.isReadable()) {
             int rc = sock.read(incomingBuffer);
             if (rc < 0) {
                 handleFailedRead();
             }
             if (incomingBuffer.remaining() == 0) {
                 boolean isPayload;
                 // 新请求 读取请求长度
                 if (incomingBuffer == lenBuffer) { // start of next request
                     incomingBuffer.flip();
                     isPayload = readLength(k);
                     incomingBuffer.clear();
                 } else {
                     // continuation
                     isPayload = true;
                 }
                 if (isPayload) { // not the case for 4letterword
                     // 读取请求内容
                     readPayload();
                 } else {
                     // four letter words take care
                     // need not do anything else
                     return;
                 }
             }
         }
         if (k.isWritable()) {
             handleWrite(k);

             if (!initialized && !getReadInterest() && !getWriteInterest()) {
                 throw new CloseRequestException("responded to info probe", DisconnectReason.INFO_PROBE);
             }
         }
     } catch (CancelledKeyException e) {
         LOG.warn("CancelledKeyException causing close of session: 0x{}", Long.toHexString(sessionId));

         LOG.debug("CancelledKeyException stack trace", e);

         close(DisconnectReason.CANCELLED_KEY_EXCEPTION);
     } catch (CloseRequestException e) {
         // expecting close to log session closure
         close();
     } catch (EndOfStreamException e) {
         LOG.warn("Unexpected exception", e);
         // expecting close to log session closure
         close(e.getReason());
     } catch (ClientCnxnLimitException e) {
         // Common case exception, print at debug level
         ServerMetrics.getMetrics().CONNECTION_REJECTED.add(1);
         LOG.warn("Closing session 0x{}", Long.toHexString(sessionId), e);
         close(DisconnectReason.CLIENT_CNX_LIMIT);
     } catch (IOException e) {
         LOG.warn("Close of session 0x{}", Long.toHexString(sessionId), e);
         close(DisconnectReason.IO_EXCEPTION);
     }
 }


private void readPayload() throws IOException, InterruptedException, ClientCnxnLimitException {
    if (incomingBuffer.remaining() != 0) { // have we read length bytes?
        int rc = sock.read(incomingBuffer); // sock is non-blocking, so ok
        if (rc < 0) {
            handleFailedRead();
        }
    }

    if (incomingBuffer.remaining() == 0) { // have we read length bytes?
        incomingBuffer.flip();
        packetReceived(4 + incomingBuffer.remaining());
        if (!initialized) {C
            // 处理连接请求 连接初始化
            readConnectRequest();
        } else {
            // 读取请求
            readRequest();
        }
        lenBuffer.clear();
        incomingBuffer = lenBuffer;
    }
}
```



## Client

1. 

