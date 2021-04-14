# New

## steps

```flow
start=>start: start
checkClassIsLoadedOrNot=>operation: check if the class is loaded
loadClass=>operation: load class
IsLoaded=>condition: is loaded or not?
allocMemory=>operation: alloc memory
init=>operation: init
setObjHeader=>operation: set object header
end=>end

start->checkClassIsLoadedOrNot->IsLoaded
IsLoaded(yes)->allocMemory
IsLoaded(no)->loadClass->allocMemory->init->setObjHeader
```



