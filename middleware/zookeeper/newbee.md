# Newbee

## Download

```bash
wget https://ftp.wayne.edu/apache/zookeeper/zookeeper-3.7.0/apache-zookeeper-3.7.0-bin.tar.gz --directory-prefix ~/Middleware

cd ~/Middleware
tar xvf apache-zookeeper-3.7.0-bin.tar.gz
```

## Command

* Create

  ```bash
  # create normal persistent node
  
  # must start with / this will add a root node yu
  create /yu 123
  # add a xx node to yu node
  create /yu/xx 456
  
  # create sequence-persistent(顺序持久化) node 
  create -s /yu/sequence-node_   # output : Created /yu/sequence-node_0000000001
  create -s /yu/sequence-node/   # error : Node does not exist: /yu/sequence-node/
  
  # create ephemeral-persistent(临时持久化) node which die with session
  # when client leaved the node will be deleted 
  # this node is not allowed to have chile node 
  create -e /ee
  ```

* Ls

  ```bash
  # list the root node 
  ls / 
  
  # list the node belong yu node 
  ls /yu
  ```

* Get

  ```bash
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
  cteate /yu/xx 456
  delete /yu
  ```

* Set

  ```bash
  create /yu 123
  create /yu/xx 456
  
  set /yu ddd
  get /yu # output: ddd
  ```

  

* 







