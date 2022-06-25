# Newbee

## Download

```bash
wget https://ftp.wayne.edu/apache/zookeeper/zookeeper-3.7.0/apache-zookeeper-3.7.0-bin.tar.gz --directory-prefix ~/Middleware
jj
cd ~/Middleware
tar xvf apache-zookeeper-3.7.0-bin.tar.gz
```

## Command

* Create

  ```bash
  create [-s] [-e] [-c] [-t ttl] path [data] [acl]
  
  # create normal persistent node
  # must start with / this will add a root node yu
  create /yu 123
  # add a xx node to yu node
  create /yu/xx 456
  
  # create persistent-sequence(顺序持久化) node 
  create -s /yu/sequence-node_   # output : Created /yu/sequence-node_0000000001
  create -s /yu/sequence-node/   # error : Node does not exist: /yu/sequence-node/
  
  # create ephemeral(临时) node which die with session
  # when client leaved the node will be deleted 
  # this node is not allowed to have chile node 
  create -e /ee
  
  # create ephemeral-persistent(临时顺序) node
  create -e -s /es
  
  # create container node
  # When the last child of a container is deleted, the container becomes a candidate to be deleted by the server at some point in the future.
  create -c /c
  
  
  # create ttl node 
  # If the znode is not modified within the TTL and has no children it will become a candidate to be deleted by the server at some point in the future.
  create -t 3000 /ttl
  
  # create persistent-sequence-with-ttl(顺序持久化 ttl) Node
  # to use this type of node you need to add property to server jvm 
  # -Dzookeeper.extendedTypesEnabled=true
  create -s -t 3000 /st
  ```

* Ls

  ```bash
  ls [-s] [-w] [-R] path     
  # -s stat
  # -w watch
  # -R rescursion
  
  # list the root node 
  ls / 
  
  # list the node belong yu node 
  ls /yu
  ```

* Get

  ```bash
  # get [-s] [-w] path
  
  # must use full path
  get /yu
  
  get /yu/xx
  ```

* Delete/Delete all

  ```bash
  # delete must be empty 
  delete /yu  # error : Node not empty
  delete /yu/xx
  
  # deleteall can delete a node not empty
  create /yu/xx 456
  deleteall /yu
  ```

* Set

  ```bash
  create /yu 123
  create /yu/xx 456
  
  set /yu ddd
  get /yu # output: ddd
  ```

  

* 



# Reference

* [https://zookeeper.apache.org/doc/r3.7.0/index.html](https://zookeeper.apache.org/doc/r3.7.0/index.html)



