java-large-file-uploader-demo
=============================

上传大文件java demo。

1，项目调研

因为需要研究下断点上传的问题。找了很久终于找到一个比较好的项目。

在GoogleCode上面，代码弄下来超级不方便，还是配置hosts才好，把代码重新上传到了github上面。


https://github.com/freewebsys/java-large-file-uploader-demo

效果：

上传中，显示进度，时间，百分比。


点击【Pause】暂停，点击【Resume】继续。


2，代码分析

原始项目：

https://code.google.com/p/java-large-file-uploader/

这个项目最后更新的时间是 2012 年，项目进行了封装使用最简单的方法实现了http的断点上传。

因为html5 里面有读取文件分割文件的类库，所以才可以支持断点上传，所以这个只能在html5 支持的浏览器上面展示。

同时，在js 和 java 同时使用 cr32 进行文件块的校验，保证数据上传正确。

代码在使用了最新的servlet 3.0 的api，使用了异步执行，监听等方法。

上传类UploadServlet
[java] view plain copy
print?在CODE上查看代码片派生到我的代码片

    @Component("javaLargeFileUploaderServlet")  
    @WebServlet(name = "javaLargeFileUploaderServlet", urlPatterns = { "/javaLargeFileUploaderServlet" })  
    public class UploadServlet extends HttpRequestHandlerServlet  
            implements HttpRequestHandler {  
      
        private static final Logger log = LoggerFactory.getLogger(UploadServlet.class);  
      
        @Autowired  
        UploadProcessor uploadProcessor;  
      
        @Autowired  
        FileUploaderHelper fileUploaderHelper;  
      
        @Autowired  
        ExceptionCodeMappingHelper exceptionCodeMappingHelper;  
      
        @Autowired  
        Authorizer authorizer;  
      
        @Autowired  
        StaticStateIdentifierManager staticStateIdentifierManager;  
      
      
      
        @Override  
        public void handleRequest(HttpServletRequest request, HttpServletResponse response)  
                throws IOException {  
            log.trace("Handling request");  
      
            Serializable jsonObject = null;  
            try {  
                // extract the action from the request  
                UploadServletAction actionByParameterName =  
                        UploadServletAction.valueOf(fileUploaderHelper.getParameterValue(request, UploadServletParameter.action));  
      
                // check authorization  
                checkAuthorization(request, actionByParameterName);  
      
                // then process the asked action  
                jsonObject = processAction(actionByParameterName, request);  
      
      
                // if something has to be written to the response  
                if (jsonObject != null) {  
                    fileUploaderHelper.writeToResponse(jsonObject, response);  
                }  
      
            }  
            // If exception, write it  
            catch (Exception e) {  
                exceptionCodeMappingHelper.processException(e, response);  
            }  
      
        }  
      
      
        private void checkAuthorization(HttpServletRequest request, UploadServletAction actionByParameterName)  
                throws MissingParameterException, AuthorizationException {  
      
            // check authorization  
            // if its not get progress (because we do not really care about authorization for get  
            // progress and it uses an array of file ids)  
            if (!actionByParameterName.equals(UploadServletAction.getProgress)) {  
      
                // extract uuid  
                final String fileIdFieldValue = fileUploaderHelper.getParameterValue(request, UploadServletParameter.fileId, false);  
      
                // if this is init, the identifier is the one in parameter  
                UUID clientOrJobId;  
                String parameter = fileUploaderHelper.getParameterValue(request, UploadServletParameter.clientId, false);  
                if (actionByParameterName.equals(UploadServletAction.getConfig) && parameter != null) {  
                    clientOrJobId = UUID.fromString(parameter);  
                }  
                // if not, get it from manager  
                else {  
                    clientOrJobId = staticStateIdentifierManager.getIdentifier();  
                }  
      
                  
                // call authorizer  
                authorizer.getAuthorization(  
                        request,  
                        actionByParameterName,  
                        clientOrJobId,  
                        fileIdFieldValue != null ? getFileIdsFromString(fileIdFieldValue).toArray(new UUID[] {}) : null);  
      
            }  
        }  
      
      
        private Serializable processAction(UploadServletAction actionByParameterName, HttpServletRequest request)  
                throws Exception {  
            log.debug("Processing action " + actionByParameterName.name());  
      
            Serializable returnObject = null;  
            switch (actionByParameterName) {  
                case getConfig:  
                    String parameterValue = fileUploaderHelper.getParameterValue(request, UploadServletParameter.clientId, false);  
                    returnObject =  
                            uploadProcessor.getConfig(  
                                    parameterValue != null ? UUID.fromString(parameterValue) : null);  
                    break;  
                case verifyCrcOfUncheckedPart:  
                    returnObject = verifyCrcOfUncheckedPart(request);  
                    break;  
                case prepareUpload:  
                    returnObject = prepareUpload(request);  
                    break;  
                case clearFile:  
                    uploadProcessor.clearFile(UUID.fromString(fileUploaderHelper.getParameterValue(request, UploadServletParameter.fileId)));  
                    break;  
                case clearAll:  
                    uploadProcessor.clearAll();  
                    break;  
                case pauseFile:  
                    List<UUID> uuids = getFileIdsFromString(fileUploaderHelper.getParameterValue(request, UploadServletParameter.fileId));  
                    uploadProcessor.pauseFile(uuids);  
                    break;  
                case resumeFile:  
                    returnObject =  
                            uploadProcessor.resumeFile(UUID.fromString(fileUploaderHelper.getParameterValue(request, UploadServletParameter.fileId)));  
                    break;  
                case setRate:  
                    uploadProcessor.setUploadRate(UUID.fromString(fileUploaderHelper.getParameterValue(request, UploadServletParameter.fileId)),  
                            Long.valueOf(fileUploaderHelper.getParameterValue(request, UploadServletParameter.rate)));  
                    break;  
                case getProgress:  
                    returnObject = getProgress(request);  
                    break;  
            }  
            return returnObject;  
        }  
      
      
        List<UUID> getFileIdsFromString(String fileIds) {  
            String[] splittedFileIds = fileIds.split(",");  
            List<UUID> uuids = Lists.newArrayList();  
            for (int i = 0; i < splittedFileIds.length; i++) {  
                uuids.add(UUID.fromString(splittedFileIds[i]));  
            }   
            return uuids;  
        }  
      
      
        private Serializable getProgress(HttpServletRequest request)  
                throws MissingParameterException {  
            Serializable returnObject;  
            String[] ids =  
                    new Gson()  
                            .fromJson(fileUploaderHelper.getParameterValue(request, UploadServletParameter.fileId), String[].class);  
            Collection<UUID> uuids = Collections2.transform(Arrays.asList(ids), new Function<String, UUID>() {  
      
                @Override  
                public UUID apply(String input) {  
                    return UUID.fromString(input);  
                }  
      
            });  
            returnObject = Maps.newHashMap();  
            for (UUID fileId : uuids) {  
                try {  
                    ProgressJson progress = uploadProcessor.getProgress(fileId);  
                    ((HashMap<String, ProgressJson>) returnObject).put(fileId.toString(), progress);  
                }  
                catch (FileNotFoundException e) {  
                    log.debug("No progress will be retrieved for " + fileId + " because " + e.getMessage());  
                }  
            }  
            return returnObject;  
        }  
      
      
        private Serializable prepareUpload(HttpServletRequest request)  
                throws MissingParameterException, IOException {  
      
            // extract file information  
            PrepareUploadJson[] fromJson =  
                    new Gson()  
                            .fromJson(fileUploaderHelper.getParameterValue(request, UploadServletParameter.newFiles), PrepareUploadJson[].class);  
      
            // prepare them  
            final HashMap<String, UUID> prepareUpload = uploadProcessor.prepareUpload(fromJson);  
      
            // return them  
            return Maps.newHashMap(Maps.transformValues(prepareUpload, new Function<UUID, String>() {  
      
                public String apply(UUID input) {  
                    return input.toString();  
                };  
            }));  
        }  
      
      
        private Boolean verifyCrcOfUncheckedPart(HttpServletRequest request)  
                throws IOException, MissingParameterException, FileCorruptedException, FileStillProcessingException {  
            UUID fileId = UUID.fromString(fileUploaderHelper.getParameterValue(request, UploadServletParameter.fileId));  
            try {  
                uploadProcessor.verifyCrcOfUncheckedPart(fileId,  
                        fileUploaderHelper.getParameterValue(request, UploadServletParameter.crc));  
            }  
            catch (InvalidCrcException e) {  
                // no need to log this exception, a fallback behaviour is defined in the  
                // throwing method.  
                // but we need to return something!  
                return Boolean.FALSE;  
            }  
            return Boolean.TRUE;  
        }  
    }  


异步上传UploadServletAsync

[java] view plain copy
print?在CODE上查看代码片派生到我的代码片

    @Component("javaLargeFileUploaderAsyncServlet")  
    @WebServlet(name = "javaLargeFileUploaderAsyncServlet", urlPatterns = { "/javaLargeFileUploaderAsyncServlet" }, asyncSupported = true)  
    public class UploadServletAsync extends HttpRequestHandlerServlet  
            implements HttpRequestHandler {  
      
        private static final Logger log = LoggerFactory.getLogger(UploadServletAsync.class);  
      
        @Autowired  
        ExceptionCodeMappingHelper exceptionCodeMappingHelper;  
      
        @Autowired  
        UploadServletAsyncProcessor uploadServletAsyncProcessor;  
          
        @Autowired  
        StaticStateIdentifierManager staticStateIdentifierManager;  
      
        @Autowired  
        StaticStateManager<StaticStatePersistedOnFileSystemEntity> staticStateManager;  
      
        @Autowired  
        FileUploaderHelper fileUploaderHelper;  
      
        @Autowired  
        Authorizer authorizer;  
      
        /** 
         * Maximum time that a streaming request can take.<br> 
         */  
        private long taskTimeOut = DateUtils.MILLIS_PER_HOUR;  
      
      
        @Override  
        public void handleRequest(final HttpServletRequest request, final HttpServletResponse response)  
                throws ServletException, IOException {  
      
            // process the request  
            try {  
      
                //check if uploads are allowed  
                if (!uploadServletAsyncProcessor.isEnabled()) {  
                    throw new UploadIsCurrentlyDisabled();  
                }  
                  
                // extract stuff from request  
                final FileUploadConfiguration process = fileUploaderHelper.extractFileUploadConfiguration(request);  
      
                log.debug("received upload request with config: "+process);  
      
                // verify authorization  
                final UUID clientId = staticStateIdentifierManager.getIdentifier();  
                authorizer.getAuthorization(request, UploadServletAction.upload, clientId, process.getFileId());  
      
                //check if that file is not paused  
                if (uploadServletAsyncProcessor.isFilePaused(process.getFileId())) {  
                    log.debug("file "+process.getFileId()+" is paused, ignoring async request.");  
                    return;  
                }  
                  
                // get the model  
                StaticFileState fileState = staticStateManager.getEntityIfPresent().getFileStates().get(process.getFileId());  
                if (fileState == null) {  
                    throw new FileNotFoundException("File with id " + process.getFileId() + " not found");  
                }  
      
                // process the request asynchronously  
                final AsyncContext asyncContext = request.startAsync();  
                asyncContext.setTimeout(taskTimeOut);  
      
      
                // add a listener to clear bucket and close inputstream when process is complete or  
                // with  
                // error  
                asyncContext.addListener(new UploadServletAsyncListenerAdapter(process.getFileId()) {  
      
                    @Override  
                    void clean() {  
                        log.debug("request " + request + " completed.");  
                        // we do not need to clear the inputstream here.  
                        // and tell processor to clean its shit!  
                        uploadServletAsyncProcessor.clean(clientId, process.getFileId());  
                    }  
                });  
      
                // then process  
                uploadServletAsyncProcessor.process(fileState, process.getFileId(), process.getCrc(), process.getInputStream(),  
                        new WriteChunkCompletionListener() {  
      
                            @Override  
                            public void success() {  
                                asyncContext.complete();  
                            }  
      
      
                            @Override  
                            public void error(Exception exception) {  
                                // handles a stream ended unexpectedly , it just means the user has  
                                // stopped the  
                                // stream  
                                if (exception.getMessage() != null) {  
                                    if (exception.getMessage().equals("Stream ended unexpectedly")) {  
                                        log.warn("User has stopped streaming for file " + process.getFileId());  
                                    }  
                                    else if (exception.getMessage().equals("User cancellation")) {  
                                        log.warn("User has cancelled streaming for file id " + process.getFileId());  
                                        // do nothing  
                                    }  
                                    else {  
                                        exceptionCodeMappingHelper.processException(exception, response);  
                                    }  
                                }  
                                else {  
                                    exceptionCodeMappingHelper.processException(exception, response);  
                                }  
      
                                asyncContext.complete();  
                            }  
      
                        });  
            }  
            catch (Exception e) {  
                exceptionCodeMappingHelper.processException(e, response);  
            }  
      
        }  
      
    }  




3，请求流程图：

主要思路就是将文件切分，然后分块上传。
