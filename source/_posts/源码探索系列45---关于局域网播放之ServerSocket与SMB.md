title: 源码探索系列45---关于局域网播放与ServerSocket
date: 2016-10-08 15:25
tags: [android,socket]
categories: android

------

最近需要做一个局域网视频播放的功能，用到两个第三方库，然后遇到了传送失败，速度为0 的问题，只好看下整个库的源码，改下bug，在这里记录下整个逻辑的流程笔记。


整个局域网播放流畅主要是下面的流程


	客户端 <--- (socket通讯)--->服务器<---(smb协议)--->某台局域网电脑


我们的安卓手机如果要播放局域网某台电脑上的视频，我们需要起一个http服务器，由这个服务器和播放器做通讯，根据播放器 发来的请求头的`range`（类似断点续成的原理），来加载特定的数据给播放器。
而服务器的数据是从某台局域网电脑上读取的啊，在windows上，有`smb`协议负责处理这部分内容，我们靠着这个smb协议去链接 `数据源` ，即某台有我们想要播放的视频的那台电脑。去加载播放器请求的内容，然后扔回给播放器。

大致的流程就是上面这样。至于具体的代码细节，在下面给出demo

<!--more-->



# 起航


首先来看下我们的http服务器部分的内容，

	SmbOverHttpService httpService = SmbOverHttpService.getInstance();
    httpService.start();

当然开头很简单，就是启动他，背后具体是下面这样

	public static int mBindPort = 0;
    public static String mDefalutServerIP = "127.0.0.1";
    private int default_port=5356;


	public boolean start() {
        if (isServing()) {
            return true;
        }
        int port = defalut_port;
        //尝试去绑定一个端口，因为有些端口会被霸用，
        //所以这里写了一个循环去试
        while (mHttpServer == null) {
            HTTPServer httpServer = new HTTPServer(mDefalutServerIP, port);
            try {
                httpServer.start();
                mHttpServer=httpServer;
            } catch (IOException e) {
                e.printStackTrace();
            }

            if (mHttpServer != null) {
                mBindPort = port;
            }
            ++port;//绑定默认端口失败就一直加，然后继续尝试绑定。直到成功
        }
         return isServing();
      }
	
	    public boolean isServing() {
	        return mHttpServer != null;
	    }

下面我们来看下这个 HTTPServer的内容
## httpServer

	
	public class HTTPServer extends NanoHTTPD {

	      public HTTPServer(String addr, int port) {
	        super(addr, port);
	     }
			
		@Override
	    public Response serve(IHTTPSession session) {
 
 	        Method requestMethod = session.getMethod();
 	        String requestUri = session.getUri();

	        if (requestMethod == Method.GET && 
		        requestUri.startsWith(SAMBA_PREFIX)) {
	            
	            return serveSamba(session);
	        }
	        
	        return newFixedLengthResponse(Status.FORBIDDEN, 
									        "text/html", "Forbidden");
	    }
		
	   private Response serveSamba(IHTTPSession session) {

			...something
			／／在这里我们去处理播放器发来的请求，通过smb读取视频的数据返回。	
		}
			
		...
		
	}

我们的server是继承[NanoHTTPD](https://github.com/NanoHttpd/nanohttpd)的，他是一个非常简易的http协议实现，所以我们继承后需要处理的就是实现`server()`函数，去分不同的情况作处理。目前我们只对get请求的读取smb内容作处理。具体的处理我们现在先不看，继续去看下绑定时候的`start（）`函数背后的内容

	public void start() throws IOException {
        start(NanoHTTPD.SOCKET_READ_TIMEOUT);
    }
    
    public void start(final int timeout) throws IOException {
        start(timeout, true);
    }
    
	public void start(final int timeout, boolean daemon) throws IOException {
    
        this.myServerSocket = this.getServerSocketFactory().create();
        this.myServerSocket.setReuseAddress(true);

		//创建的这个新的ServerRunnable对象，
		//通过名字就可以猜到是一个runnable对象，要不然也不会扔到thread做参数了
        ServerRunnable serverRunnable = createServerRunnable(timeout);
        this.myThread = new Thread(serverRunnable);
        
        this.myThread.setDaemon(daemon);
        this.myThread.setName("NanoHttpd Main Listener");
        this.myThread.start();
        while (!serverRunnable.hasBinded() &&
		         serverRunnable.getBindException() == null) {
		         
            try {
                Thread.sleep(10L);
            } catch (Throwable e) {
                // on android this may not be allowed, that's why we
                // catch throwable the wait should be very short because we are
                // just waiting for the bind of the socket
            }
            
        }
        if (serverRunnable.getBindException() != null) {
            throw serverRunnable.getBindException();
        }
    }

上面整个背后的套路，其实就是在`ServerSocket.accept()` 得到一个socket后，开一个线程去处理，[具体看这份入门demo](http://blog.csdn.net/yangyi22/article/details/7523968)。

然后我们来看下这个创建一个runnable的内容

### createServerRunnable

	 protected ServerRunnable createServerRunnable(final int timeout) {
        return new ServerRunnable(this, timeout);
    }
	
 我们来看下这个serverRunnable内容，整个类不大，就整个粘上来了。
	
	public class ServerRunnable implements Runnable {

	    private NanoHTTPD httpd;
	    private final int timeout;
	    private IOException bindException;
	    private boolean hasBinded = false;

	    public ServerRunnable(NanoHTTPD httpd, int timeout) {
	        this.httpd = httpd;
	        this.timeout = timeout;
	    }
	
	    @Override
	    public void run() {
	        try {
		        //显然我们是有hostname=“127.0.0.0”的
		         //就在这一行，我们调用了ServerSocket去绑定hostname+port
	              httpd.getMyServerSocket().bind(httpd.hostname != null ?
	              new InetSocketAddress(httpd.hostname, httpd.myPort) :
	              new InetSocketAddress(httpd.myPort));
	               
	            hasBinded = true;
	        } catch (IOException e) {
		        //如果绑定失败，就保存下这个e，然后上面就根据这个做判断看是否绑定成功。
	            this.bindException = e;
	            return;
	        }

	        do {
	            try {
		            //前面绑定成功后，就调用ServerSocket的accept函数去等客户端消息来
	                Socket finalAccept = httpd.getMyServerSocket().accept();
	                if (this.timeout > 0) {
	                    finalAccept.setSoTimeout(this.timeout);
	                }
	                //有了消息，就拿inputStream给开一条新的线程去处理这次通讯。
	                InputStream inputStream = finalAccept.getInputStream();
	                httpd.asyncRunner.exec(httpd.createClientHandler(
							                finalAccept, inputStream));
							                
	            } catch (IOException e) {
	                log(Level.FINE, "Communication with the client broken", e);
	            }
	        } while (!httpd.getMyServerSocket().isClosed());
	        //这里写了一个循环在这里，直到服务器关了才不再监听消息。     
	    }
	
	    public IOException getBindException() {
	        return bindException;
	    }
	
	    public boolean hasBinded() {
	        return hasBinded;
	    }
	}

我们看下run函数里面，拿从`ServerSocket.accept()`得到的`socket`处理部分，就是下面这句

	  httpd.asyncRunner.exec(httpd.createClientHandler(
							                finalAccept, inputStream));

		        
	 ClientHandler createClientHandler(Socket socket,InputStream inStream) {
        return new ClientHandler(this, inStream, socket);
	  }
  就是创建一个新的runnable去处理这次的客户请求

然后这个asyncRunner.exec 就是一个套壳的thread函数，我们截取了核心的代码段。
	
	@Override
    public void exec(ClientHandler clientHandler) {
        ...
        createThread(clientHandler).start();
    }
    
    protected Thread createThread(ClientHandler clientHandler){
    	Thread t = new Thread(clientHandler);
        t.setDaemon(true);
        t.setName("NanoHttpd Request Processor (#" + this.requestCount + ")");
        return t;
    }    

所以我们还是去看下这个ClientHandler的run函数的内容吧，这个才是全部的重点内容

### ClientHandler
 
	/**
	 * The runnable that will be used for every new client connection.
	 */
	public class ClientHandler implements Runnable {

	    private final NanoHTTPD httpd;
		private final InputStream inputStream;
	    private final Socket acceptSocket;
	
	    public ClientHandler(NanoHTTPD httpd, 
			  InputStream inputStream, Socket acceptSocket) {
	       
	        this.httpd = httpd;
	        this.inputStream = inputStream;
	        this.acceptSocket = acceptSocket;
	    }
	
	    public void close() {
	        NanoHTTPD.safeClose(this.inputStream);
	        NanoHTTPD.safeClose(this.acceptSocket);
	    }
	
	    @Override
	    public void run() {
	        OutputStream outputStream = null;
	        try {
	            outputStream = this.acceptSocket.getOutputStream();
	            
	            ITempFileManager tempFileManager = 
		            httpd.getTempFileManagerFactory().create();
	            
	            HTTPSession session = new HTTPSession(httpd, 
		            tempFileManager, this.inputStream,outputStream, 
		            this.acceptSocket.getInetAddress());

				//这段就是构造了一个session，接着掉哟功能execute，
				//估计会回调到我们开头处理请求的server()
	            while (!this.acceptSocket.isClosed()) {
	                session.execute();
	            }
	            
	        } catch (Exception e) {
	            // When the socket is closed by the client,
	            // we throw our own SocketException to 
	            // break the "keep alive" loop above. If
	            // the exception was anything other than 
	            //the expected SocketException OR a
	            // SocketTimeoutException, print the  stacktrace
	          
	            if (!(e instanceof SocketException &&
		             "NanoHttpd Shutdown".equals(e.getMessage())) &&
		              !(e instanceof SocketTimeoutException)) {
	              
	              log(Level.SEVERE, "Communication with the client broken, 
									         or an bug in the handler code", e);
									             
	            }
	            
	        } finally {
	            NanoHTTPD.safeClose(outputStream);
	            NanoHTTPD.safeClose(this.inputStream);
	            NanoHTTPD.safeClose(this.acceptSocket);
	            httpd.asyncRunner.closed(this);
	        }
	    }
	}


## httpSession.execute()
这个httpSession有点长，不好直接都粘贴上来，毕竟整个请求解析的实现都浓缩在这里了

	public class HTTPSession implements IHTTPSession {

		...
	
	  public HTTPSession(NanoHTTPD httpd, ITempFileManager tempFileManager,
						 InputStream inputStream, OutputStream outputStream, 
						 InetAddress inetAddress) {
	    ／／ 把构造函数贴出来，直到变量的关系，便于后面的execute函数内容的理解
        this.httpd = httpd;
        this.tempFileManager = tempFileManager;
        this.inputStream = new BufferedInputStream(inputStream,
							         HTTPSession.BUFSIZE);
        this.outputStream = outputStream;
        this.remoteIp = inetAddress.isLoopbackAddress() ||
				        inetAddress.isAnyLocalAddress() ?
						        "127.0.0.1" : 
						       inetAddress.getHostAddress().toString();
						       
        this.remoteHostname = inetAddress.isLoopbackAddress() || 
						        inetAddress.isAnyLocalAddress() ?
								        "localhost" : 
									    inetAddress.getHostName().toString();
									    
        this.headers = new HashMap<String, String>();
    }
	
	...
	
	 @Override
     public void execute() throws IOException {
        Response r = null;
        try {
        
            byte[] buf = new byte[HTTPSession.BUFSIZE];
            this.splitbyte = 0;
            this.rlen = 0;
            int read = -1;
            this.inputStream.mark(HTTPSession.BUFSIZE);
            try {
                read = this.inputStream.read(buf, 0, HTTPSession.BUFSIZE);
            } catch (SSLException e) {
                throw e;
            } catch (IOException e) {
                NanoHTTPD.safeClose(this.inputStream);
                NanoHTTPD.safeClose(this.outputStream);
                throw new SocketException("NanoHttpd Shutdown");
            }
            
			//读取数据到buf部分
            if (read == -1) {
                // socket was been closed
                NanoHTTPD.safeClose(this.inputStream);
                NanoHTTPD.safeClose(this.outputStream);
                throw new SocketException("NanoHttpd Shutdown");
            }
            while (read > 0) {
                this.rlen += read;
                this.splitbyte = findHeaderEnd(buf, this.rlen);
                if (this.splitbyte > 0) {
                    break;
                }
                read = this.inputStream.read(buf, this.rlen, 
							                HTTPSession.BUFSIZE - this.rlen);
            }

            if (this.splitbyte < this.rlen) {
                this.inputStream.reset();
                this.inputStream.skip(this.splitbyte);
            }
            
			／／下面开始是🐷部解析
            this.parms = new HashMap<String, List<String>>();
            if (null == this.headers) {
                this.headers = new HashMap<String, String>();
            } else {
                this.headers.clear();
            }
            // Create a BufferedReader for parsing the header.
            BufferedReader hin = new BufferedReader(new InputStreamReader(
										            new ByteArrayInputStream(buf, 
											            0, this.rlen)));
											            
            // Decode the header into parms and header java properties
            Map<String, String> pre = new HashMap<String, String>();
            decodeHeader(hin, pre, this.parms, this.headers);
			//具体怎么解析我们这次不关心，不看🙈 
			
            if (null != this.remoteIp) {
                this.headers.put("remote-addr", this.remoteIp);
                this.headers.put("http-client-ip", this.remoteIp);
            }

			//根据解析的结果，初始化这个session
			//还是另起一个函数来处理吧，不要都扔在这里，这个函数已经很长了
            this.method = Method.lookup(pre.get("method"));           
            this.uri = pre.get("uri");
            this.cookies = new CookieHandler(this.headers);
            String connection = this.headers.get("connection");
            boolean keepAlive = "HTTP/1.1".equals(protocolVersion)
					             && (connection == null || 
							         !connection.matches("(?i).*close.*"));


			 //我们比较关心的内容来了，通过前面的一堆解析，构造好必备的session信息，
			 //就去回调我们的serve函数，我们去具体的处理这个请求
            r = httpd.serve(this);
	           //处理完生成一个Respone对象，最后我们在send发送给客户端
             
            if (r == null) {
                throw new NanoHTTPD.ResponseException(Status.INTERNAL_ERROR, 
	                "SERVER INTERNAL ERROR: Serve() returned a null response.");
            } else {
                String acceptEncoding = this.headers.get("accept-encoding");
                this.cookies.unloadQueue(r);
                r.setRequestMethod(this.method);
                r.setGzipEncoding(httpd.useGzipWhenAccepted(r) 
					              && acceptEncoding != null
				                  && acceptEncoding.contains("gzip"));
				                  
                r.setKeepAlive(keepAlive);
                r.send(this.outputStream);
                //会送消息给客户端
            }
            
            if (!keepAlive || r.isCloseConnection()) {
                throw new SocketException("NanoHttpd Shutdown");
            }
            
        } catch (Exception e) {
	        ...
        } finally {
            NanoHTTPD.safeClose(r);
            this.tempFileManager.clear();
        }
    }

看我这上面的一大串，我们来缕清下思路，主要就是在accept得到socket后，根据socket得到的inputStream去读客户端发送来的内容，接着就是解析，解析好后是去serve函数处理请求，处理得到一个response结果再回传给客户端。

明白上面的过程，我们现在就可以去看下怎么处理这个请求
我们直接就来看下serveSamba的内容吧

##  serveSamba


	 //对于处理smb协议的核心处理部分，
    //跑到这里根据get请求头的range信息，从smbFile对象拿信息返回
    private Response serveSamba(IHTTPSession session) {
     
        String reqUri = session.getUri();
        String base64EncodeSmbUri = reqUri.substring(SAMBA_PREFIX.length());
        byte[] smbUriBytes = Base64.decode(base64EncodeSmbUri, Base64.URL_SAFE);
        String smbUri = new String(smbUriBytes, Charset.forName("utf8"));
        

         
        InputStream smbFileInputStream = null;
        long fileLength = 0;
        String filename = null; 
	    SmbFile smbFile = new SmbFile(smbUri);
	    
        if (smbFile!=null && !smbFile.exists() || !smbFile.isFile()) {
            return newFixedLengthResponse(Status.NOT_FOUND, 
				              "text/html", "Not Found");
        }
        
        fileLength = smbFile.length();
        filename = smbFile.getName();
        smbFileInputStream = smbFile.getInputStream();
       
		...
				
        boolean releaseSmbFileInputStream = true;
        try {
            Map<String, String> reqHeader = session.getHeaders();
            boolean hasHeaderRange = reqHeader.containsKey("range");
 
            long rangeBegin = 0;
            long rangeEnd = -1;
            if (hasHeaderRange) {
	            //对于请求带range的，就解析下
                 String rangeValue = reqHeader.get("range");
                 boolean parseRangeSuccess = false;
                if (rangeValue.startsWith("bytes=")) {
                    int minus = rangeValue.indexOf('-');
                    if (minus > "bytes=".length()) {                       
                       rangeBegin = Long.parseLong(
		                        rangeValue.substring("bytes=".length(), minus));
                        rangeEnd = Long.parseLong(
		                        rangeValue.substring(minus + 1));                         
                        parseRangeSuccess = true;
                    }
                }
                
				...
            }

			//获取请求文件的类型avi，mp4等
            String mimeType = null;
            if (filename != null) {
                int pos = filename.lastIndexOf('.');
                if (pos > 0) {
                    mimeType = fileExtNameToMIMEType(
			                    filename.substring(pos + 1));
                }
            }

			//对于请求没带range的，就是默认的[0-fileLength]
            Response res = null;
            if (rangeEnd < 0) {
                rangeEnd = (fileLength > 0 ? fileLength - 1 : 0);
            }
            
            
            long resContentLength = rangeEnd - rangeBegin + 1;
            if (hasHeaderRange) {
	            //下面根据请求的range去返回对应的数据给客户端
	            smbFileInputStream.skip(rangeBegin);
	            
                res = newFixedLengthResponse(Status.PARTIAL_CONTENT, mimeType, 
						                smbFileInputStream, resContentLength);
                res.addHeader("Accept-Ranges", "bytes");
                res.addHeader("Content-Range", 
				                "bytes " + rangeBegin + "-" + 
				                rangeEnd + "/" + fileLength);
            } else {//不加range的就整个流送回去
                res = newFixedLengthResponse(Status.OK, mimeType, 
				                smbFileInputStream, resContentLength);
            }

            res.addHeader("Content-Length", "" + resContentLength);
            releaseSmbFileInputStream = false;
            return res;//到此顺利的构造好一个Respone，可以返回给客户端了
            
         }catch (Exception e){
            Log.i(TAG, "serveSamba: try read fail -> "+e.toString());
            return newFixedLengthResponse(Status.NOT_FOUND, 
							            "text/html", "Not Found");
		 }finally {
            if (releaseSmbFileInputStream && smbFileInputStream != null) {
                smbFileInputStream.close();//最后记得关闭文件流！
            }
            
        }
    }

看完构造response，这部分就是和局域网的数据沟通部分内容，很简单的感觉不是吗？。
一个封装好了的SmbFile对象，曾经自己写了一个类似的，实在是不堪重负🙈最后找这个第三方包


## respone.send

我们看最后的送回去给客户端的部分

	  /**
     * Sends given response to the socket.
     */
    public void send(OutputStream outputStream) {
    
	    //下面一大串就是构造头部内容，为何不单独一个函数里面...
        SimpleDateFormat gmtFrmt = new SimpleDateFormat(
			        "E, d MMM yyyy HH:mm:ss 'GMT'", Locale.US);																						         
			        
        gmtFrmt.setTimeZone(TimeZone.getTimeZone("GMT"));
        
        try {
            if (this.status == null) {
                throw new Error("sendResponse(): Status can't be null.");
            }
            
            //构造一个writer实在是太长了
            PrintWriter pw = new PrintWriter(
	            new BufferedWriter(
	             new OutputStreamWriter(outputStream, 
				new ContentType(this.mimeType).getEncoding())), false);
												            
			//使用1.1协议，话说现在2.0都出来了，解决队首堵塞等问题。				            
            pw.append("HTTP/1.1 ").
	            append(this.status.getDescription()).append(" \r\n");
            
            if (this.mimeType != null) {
                printHeader(pw, "Content-Type", this.mimeType);
            }
            if (getHeader("date") == null) {
                printHeader(pw, "Date", gmtFrmt.format(new Date()));
            }
                         
            for (String cookieHeader : this.cookieHeaders) {
            	printHeader(pw, "Set-Cookie", cookieHeader);
            }
            
            if (getHeader("connection") == null) {
                printHeader(pw, "Connection", (this.keepAlive ? 
										                "keep-alive" : "close"));
            }
            
            if (getHeader("content-length") != null) {
                encodeAsGzip = false;
            }
            
             //我们是有length的，所以不适用gzip，而且不是chunkedTransfer
                        
            if (encodeAsGzip) {
                printHeader(pw, "Content-Encoding", "gzip");
                setChunkedTransfer(true);
            }
            
	            /／致于这个pending，当然在我们的情况，播放视频的背景下
	            //data为真，用contentLength
            long pending = this.data != null ? this.contentLength : 0;
            
            if (this.requestMethod != Method.HEAD && this.chunkedTransfer) {
                printHeader(pw, "Transfer-Encoding", "chunked");
            } else if (!encodeAsGzip) {
                pending = 
		                sendContentLengthHeaderIfNotAlreadyPresent(pw, pending);
            }
            
            pw.append("\r\n");
            pw.flush();
            
            sendBodyWithCorrectTransferAndEncoding(outputStream, pending);
            
            //写完flush，发送。
            outputStream.flush();
            NanoHTTPD.safeClose(this.data);
            
        } catch (IOException ioe) {
            NanoHTTPD.LOG.log(Level.SEVERE, "Could not send response to the client", ioe);
        }
    }

	private void sendBodyWithCorrectTransferAndEncoding(
				OutputStream outputStream, long pending) throws IOException {
	
        if (this.requestMethod != Method.HEAD && this.chunkedTransfer) {
            ChunkedOutputStream chunkedOutputStream = 
		            new ChunkedOutputStream(outputStream);
		            
            sendBodyWithCorrectEncoding(chunkedOutputStream, -1);
            chunkedOutputStream.finish();
        } else {
	        //走这个分支，chunkedTransfer为fasle
            sendBodyWithCorrectEncoding(outputStream, pending);
        }
    }

    private void sendBodyWithCorrectEncoding(
		    OutputStream outputStream, long pending) throws IOException {
		    
        if (encodeAsGzip) {
            GZIPOutputStream gzipOutputStream = 
		       new GZIPOutputStream(outputStream);
		            
            sendBody(gzipOutputStream, -1);
            gzipOutputStream.finish();
            
        } else {
	        //这个分支，我们不用gzip
            sendBody(outputStream, pending);
        }
    }

    /**
     * Sends the body to the specified OutputStream. 
     * The pending parameter
     * limits the maximum amounts of bytes sent unless it is -1,
     *  in which case everything is sent. 
     */
    private void sendBody(OutputStream outputStream, long pending)
	     throws IOException {
    
        long BUFFER_SIZE = 16 * 1024;
        //然后需要说下这个缓冲大小
        //根据自己实际调整下大小。
        
        byte[] buff = new byte[(int) BUFFER_SIZE];
        boolean sendEverything = pending == -1;
		//接下来这部分就是循环读数据，每次读bufferSize的大小，然后送给客户端
		//直到读完，就算结束掉这次的socket请求了。
		
        while (pending > 0 || sendEverything) {
            long bytesToRead = sendEverything ? BUFFER_SIZE : Math.min(pending, BUFFER_SIZE);
            int read = this.data.read(buff, 0, (int) bytesToRead);
            if (read <= 0) {
                break;
            }
            outputStream.write(buff, 0, read);
            if (!sendEverything) {
                pending -= read;
            }
        }
    }

好了，整个流程就这样看我了，整个demo也基本核心的都贴出来了。

看我整个流程，理论上一个视频的播放流不流畅的关键，即瓶颈我们也知道在哪里了，
一般主要耗费的时间一般都在sendBody(）这个函数，例如当你的pending就是文件大小，即你的range就是0-fileSize的时候。
那么到底要怎么改进性能呢？

改进性能是一个很大的话题，最好有一些工具帮助你去量化，测量结果。
从内存，耗时，网络带宽等等多方面考察，Android studio带的monitor工具就是一个挺好的帮手，具体见我前面的文章。

1. 播放器内核直接支持
关于局域网播放，在处理一些小视频还好，要是遇到一些超高码率的，例如那些1G大小的视频居然只有几十秒的那些。
那么假设一个中间转接的服务器是不太合适的，最好就是让播放器直接支持局域网播放。
毕竟像上面这样，读取完，再写回给客户端，客户端再播放出来，和 客户端直接播放器读取播放是不一样的。

2. 开多个请求
以上是基于每次请求的range为0-fileSize的整个文件大小问题，那么我们可以像多点续传一样，让播放器多发点请求，多线程去加数据回来，尽量把带宽都霸占满。

3. 播放内核不支持，就改进缓存大小
缓冲大小这个是很直接的因素，例如上面写死了是16kB。但最好还是动态来调可能是最好的，毕竟这个直觉上我们能够知道是受服务器的iops，网速，客户端的读取速度等多方面影响的。如果是千兆光谦，ssd盘，我们设置缓冲1kb，那就有点浪费了。所以根据实际的服务器情况，带宽，磁盘io情况来设置这个缓冲大小会更合适。当然我们看到如果把判断条件改下，是否会缩短一点点时间呢？
 
4. 改进jcifs
我们是通过jcifs这个实现smb协议的库来和局域网的电脑交互，读取数据等。那么理论上这里面必然会有一些优化的地步，致于具体的，就靠读者你自己分析了。
Linux系统默认自带的Samba，都有优化的配置文件，估计jcifs也会有，不过好像在官网没注意到有。


# ref
1. [nanoHttpd](https://github.com/NanoHttpd/nanohttpd)，http 服务器是用了这个项目的内容
2. [jcifs](https://jcifs.samba.org/)，JCIFS is an Open Source client library that implements the CIFS/SMB networking protocol in 100% Java. CIFS is the standard file sharing protocol on the Microsoft Windows platform (e.g. Map Network Drive ...). This client is used extensively in production on large Intranets.