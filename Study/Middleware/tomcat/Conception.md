# Conception

```java

Pipeline
    List<valve>;

Server
    List<Service>
Service 
    Connector
    	ProtocolHandler(eg: HTTP/1.1 Http11NioProtocol)
    		EndPoint(eg: NioEndPoint)
    List<Engine>
Engine extends Container
	List<Engine>
    Pipeline
    	Valve ....basicValve=standard***Valve
Host extends Container
	List<Context>
    Pipeline
Context extends Container, ContextBind
	List<wrapper>
    Pipeline
Wrapper extends Container  
	List<Servlet>
    Servlet
    Pipeline          standardWrapperValve -> doFilterChain -> servlet.service()
    

    
	NioEndiPoint 
    	NioSocketWrapper
    	Poller
    NioSocketWrapper extends SocketWrapperBase
    	NioChannel
    		SocketChannel
    		socketBufferHandler
    			readBuffer
    	NioOperationState (write/read)

    NioOpreationState extends OpreationState
    	buffers
    
    
    SocketProcessor  created by SocketWrapper
    ProtocolProcessor  created bt Protocol with cache
    
```