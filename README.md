#SimpleLog

##前言

说起日志，平时调试用的Log类你肯定非常熟悉，并且用着也很方便，可是当你的项目逐渐走上正轨，并且越来越完善的时候，你可能会发现之前大量的Log输出严重影响了你项目的调试，并且分散到每个没文件里的Log输出非常难以管理。

（如果你已经开始考虑日志框架的问题了，那么说明你对日志有了一定的了解，这里就不在对日志级别之类的基础知识进行描述了，）

这个时候你可能会想：我需要一个日志管理工具！

如果你从事过javaEE开发或在网上查过“日志框架”这类关键词，那么
		
		log4j+commons-logging
		slf4j + logback
这样的优秀日志框架你一定不会陌生，他们已经成为了日志框架的主流。

**但是，对于Android平台，这样的日志框架是不是太重了呢？**通常来说，在移动开发中，我们并不建议大量引入第三方类库，每个类库的引用都是要经过细致的权衡的。

其实，我们完全根据自己的需求，可以开发自己的日志框架。这真的很简单！

##开发之初

首先在开发之初，你需要知道几件事：

- 我的需求什么？
- 主流的日志框架功能有哪些？
- 别人是怎么做的？


通常来说使用日志框架的需求有这些：
1. 控制日志输出级别，开发中输出所有日志，打release包时，只输出错误或警告日志。
2. 日志的本地保存和服务器上传
3. 可控的保存或上传开关和级别
4. 不同日志不同保存方式
5. 更优的日志使用接口
6. 更优的使用性能
7. .......


主流的日志框架功能：

slf4j 或log4j的模式是 包装类（门面）+ 日志框架。上述的需求，他们基本都已经实现，并且性能还很不错。但你需要注意的是，通常使用这些框架的都是大型服务器系统类型，他们对日志的需求很高，同时也不在乎多一个外部库。对于移动开发，你可能用不到那么多的功能，比如 “门面”的作用是可以兼容不同的日志框架，移动端有必要吗？小应用android自带的java.util.logging都够了。

别人是怎么做的：

这个时候你可能特别想知道别人是怎么做的？如果没有更好的了解渠道，反编译也许帮到你。楼主反编译了3个APK，Path、QQ空间、Google+。虽然读反编译的代码很痛苦，但你真的可以看到一些大平台是如何做的。
（关于反编译的方法，其实很简单，可以参考这篇博客：http://blog.csdn.net/vipzjyno1/article/details/21039349/）


那么好吧，说了这么多，开始正题吧。下面是楼主写的日志管理框架，一个很简单的小Demo。

####注释就是最好的文档

先看代码目录
![这里写图片描述](http://img.blog.csdn.net/20151020222917701)

很简单，关键代码只有3个。

LogConfig： Log的配置信息。
LogPrint :　   Log的具体输出实现。
WLog：        Log的 “门面” ，具体使用通过调用WLog。

先说一下日志的控制逻辑：

	/**
	     * Log输出机制： 
	     *  LogLevel：Log输出级别控制
	     * if（ 要输出的日志级别  >= LogLevel ）
	     *      输出；
	     * else
	     *      不输出；
	     *
	     * Log上传和保存机制：
		* UploadSwitch： 上传保存的总开关
	     * if（UploadSwitch == false）
	     *      什么都不做
	     * else
	     *      if  （调用Log上传方法）
	     *           上传并保存
	     *      else
	     *          if（flag>=LogConfig.UploadLevel）
	     *                  上传
	     *          if(flag>=LogConfig.SaveFileLevel)
	     *                  保存
	     *
	     */

可能看的不太明白，看完代码你就懂了，很简单。

我们先看LogConfig.java

	public class LogConfig {
    /*  Log输出级别 */
    protected static int LogLevel = LogConfig.DEBUG;

    /* Log上传和保存级别 */
    protected static int UploadLevel = LogConfig.ERROR;
    protected static int SaveFileLevel = LogConfig.ERROR;
    /** Log 上传和保存的总开关，如果关了，那么无论如何都不会执行上传或保存操作**/
    protected static boolean UploadSwitch = true;


    public static final int NO_OUTPUT = 1;  //什么都不输出
    public static final int VERBOSE = 2;
    public static final int DEBUG = 3;
    public static final int INFO = 4;
    public static final int WARN =5;
    public static final int ERROR = 6;

    protected static final String DEFAULT_TAG = "COM.PATH";

    /**
     * 在赋值同时，可以做一些权限判断
     * 这里并没有权限判断的需求，所以方法里只有很简单赋值操作
     * @param level  要设定的Log输出级别
     */
    public static void setLogLevel(int level){
        LogLevel = level;
    }
    public static int getLogLevel(){
        return LogLevel;
    }
    public static void setUploadLevel(int level){
        UploadLevel = level;
    }
    public static int getUploadLevel(){
        return UploadLevel;
    }
    public static void setSaveFileLevel(int level){
        SaveFileLevel = level;
    }
    public static int getSaveFileLevel(){
        return SaveFileLevel;
    }



    public static String getLogLevelName(int flag){
        if(flag==-1)
            flag= LogLevel;
        switch (flag){
            case NO_OUTPUT:return "NO_OUTPUT";
            case VERBOSE:return "VERBOSE";
            case DEBUG:return "DEBUG";
            case INFO:return "INFO";
            case WARN:return "WARN";
            case ERROR:return "ERROR";
            default:return "NO_OUTPUT";
        }
    }
	}

在开发中，通常只需要修改这个类里面对应的级别就好了。
这里有一个小技巧。一般我们最常用的类修饰符只有public ,private。而protected一般只知道是用在类继承时的。protected还有更多的扩展使用。

会有这样的情况，类A的成员只想暴露给关系较为亲密的B和C，并不像完全公开给所有的E、F、G、R、T、Y类。设为私有，BC访问不到。设为公开，有完全暴露给所有类的。这时你可以设为protected，protected同default一样，都可以供同一个包下的类直接访问。而一般我们又并不习惯使用default（因为什么都不写感觉怪怪的），所以这时protected就是很好的选择了。善用protected！

继续。接下来看  LogPrint.java


	public class LogPrint  {
	    /** Log日志的tag String : TAG */
	    private static final String TAG = LogPrint.class.getSimpleName();
	    /**时间格式**/
	    private static final SimpleDateFormat mSimpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
	    /**设备信息**/
	    private static StringBuilder deviceMessage = null;

    /**
     * 拼接异常对象成字符串
     * @param throwable  异常对象
     * @return
     */
    protected static StringBuilder createStackTrace(Throwable throwable){
        StringBuilder builder = new StringBuilder();
        if(throwable == null)
            return builder;
        builder.append("\n        "+throwable.getClass()+":  "+throwable.getMessage());
        int length = throwable.getStackTrace().length;
        for(int i =0;i<length;i++){
            builder.append("\n\t\t"+throwable.getStackTrace()[i]);
        }
        return builder;
    }

    /**
     * 无异常对象情况下输出Log
     * @param logLevel
     * @param tagString
     * @param explainString
     */
    protected  static void println(int logLevel, String tagString ,String explainString){
        Log.println(logLevel, tagString, explainString);
    }

    /**
     * 有异常对象情况下输出Log
     * @param logLevel
     * @param tagString
     * @param explainString
     * @param throwable
     */
    protected  static void println(int logLevel, String tagString ,String explainString,Throwable throwable){
        switch (logLevel){
            case LogConfig.VERBOSE:
                Log.v(tagString,explainString,throwable); break;
            case LogConfig.DEBUG:
                Log.d(tagString,explainString,throwable); break;
            case LogConfig.INFO:
                Log.i(tagString,explainString,throwable); break;
            case LogConfig.WARN:
                Log.w(tagString,explainString,throwable); break;
            case LogConfig.ERROR:
                Log.e(tagString, explainString, throwable); break;
        }
    }

    /**
     * 生成上传服务器日志格式
     */
    protected static String createMessageToServer(Throwable throwable){
        StringBuilder stringBuilder = new StringBuilder();
        stringBuilder.append("time" + "=" + mSimpleDateFormat.format(new Date()) + "\r\n");
        //stringBuilder.append(getDeviceInfo(paramContext));

        if(throwable==null)
            return stringBuilder.toString();

        Writer mWriter = new StringWriter();
        PrintWriter mPrintWriter = new PrintWriter(mWriter);
        throwable.printStackTrace(mPrintWriter);
        // paramThrowable.printStackTrace();
        Throwable mThrowable = throwable.getCause();
        // 迭代栈队列把所有的异常信息写入writer中
        while (mThrowable != null) {
            mThrowable.printStackTrace(mPrintWriter);
            // 换行 每个个异常栈之间换行
            mPrintWriter.append("\r\n");
            mThrowable = mThrowable.getCause();
        }
        // 记得关闭
        mPrintWriter.close();
        String mResult = mWriter.toString();
        stringBuilder.append(mResult);
        return stringBuilder.toString();
    }


    protected static StringBuilder getDeviceInfo(Context paramContext){
        //设备信息为空时，读取设备信息
        if(deviceMessage==null||deviceMessage.equals("")){
            deviceMessage = new StringBuilder();
            try {
                // 获得包管理器
                PackageManager mPackageManager = paramContext.getPackageManager();
                // 得到该应用的信息，即主Activity
                PackageInfo mPackageInfo = mPackageManager.getPackageInfo(paramContext.getPackageName(), PackageManager.GET_ACTIVITIES);
                if (mPackageInfo != null) {
                    String versionName = mPackageInfo.versionName == null ? "null" : mPackageInfo.versionName;
                    String versionCode = mPackageInfo.versionCode + "";
                    deviceMessage.append("versionName" + "=" + versionName + "\r\n");
                    deviceMessage.append("versionCode" + "=" + versionCode + "\r\n");
                }
                //mLogInfo.put("ENV_CODE", PropertiesUtils.get(paramContext, "CURRENT_ENV_CODE"));
            } catch (PackageManager.NameNotFoundException e) {
                WLog.e(TAG, "获取版本信息出错", e);
            }

            Field[] mFields = Build.class.getDeclaredFields();
            // 迭代Build的字段key-value 此处的信息主要是为了在服务器端手机各种版本手机报错的原因
            for (Field field : mFields) {
                try {
                    field.setAccessible(true);
                    if(field.getName().equals("BRAND")||field.getName().equals("DEVICE")||
                            field.getName().equals("MODEL")){
                        deviceMessage.append(field.getName() + "=" + field.get("").toString() + "\r\n");
                    }
                } catch (Exception e) {
                    WLog.e(TAG, "获取设备信息出错", e);
                }
            }
        }

        if(deviceMessage!=null)
            return deviceMessage;
        else
            return new StringBuilder();
    }


    /**
     * 保存日志到文件
     * @param flag
     * @param explainString
     */
    protected static void saveLogToFile(int flag,String explainString){
        StringBuilder message = new StringBuilder();
        message.append("time" + "=" + mSimpleDateFormat.format(new Date()) + "\r\n");
        message.append(LogConfig.getLogLevelName(flag)+"------"+explainString+"\n");

        /*-----------------------------以下调用 具体保存本地方法---------------------*/
        /*-----------------------------不同应用保存方式不用，此处不具体写---------------------*/
        Log.println(LogConfig.INFO, "Log保存到文件", message.toString());
    }
    /**
     * 保存日志到文件
     * @param flag
     * @param explainString
     */
    protected static void saveLogToFile(int flag,String explainString,Throwable throwable){
        Context context = MyApplication.getInstence();
        StringBuilder message = new StringBuilder();
        message.append("time" + "=" + mSimpleDateFormat.format(new Date()) + "\r\n");
        message.append(getDeviceInfo(context));
        message.append(LogConfig.getLogLevelName(flag)+"------"+explainString+"\n");
        message.append(createMessageToServer(throwable));


        /*-----------------------------以下调用 具体保存本地方法---------------------*/
        /*-----------------------------不同应用保存方式不用，此处不具体写---------------------*/
        Log.println(LogConfig.INFO, "Log保存到文件", message.toString());

    }

    /**
     * 上传日志到服务器
     * 当没有错误信息时，你需要考虑是否需要上传设备信息。
     * @param flag
     * @param explainString
     */
    protected static void uploadToServer(int flag,String explainString){
        Context context = MyApplication.getInstence();
        StringBuilder message = new StringBuilder();
        message.append("time" + "=" + mSimpleDateFormat.format(new Date()) + "\r\n");
        message.append(getDeviceInfo(context));  //是否需要上传设备信息呢
        message.append(LogConfig.getLogLevelName(flag)+"------"+explainString+"\n");

        /*-----------------------------以下调用 具体上传服务器方法---------------------*/
        /*-----------------------------不同应用上传方式不用，此处不具体写---------------------*/
        Log.println(LogConfig.INFO, "Log上传到服务器", message.toString());
    }
    /**
     * 上传日志到服务器
     * @param flag
     * @param explainString
     */
    protected static void uploadToServer(int flag,String explainString,Throwable throwable){
        Context context = MyApplication.getInstence();
        StringBuilder message = new StringBuilder();
        message.append("time" + "=" + mSimpleDateFormat.format(new Date()) + "\r\n");
        message.append(getDeviceInfo(context));
        message.append(LogConfig.getLogLevelName(flag)+"------"+explainString+"\n");
        message.append(createMessageToServer(throwable));

        /*-----------------------------以下调用 具体上传服务器方法---------------------*/
        /*-----------------------------不同应用上传方式不用，此处不具体写---------------------*/
        Log.println(LogConfig.INFO,"Log上传到服务器",message.toString());
    }
}

具体说明，注释已经写得很清楚了。唯一需要注意的是，因为每个应用的保存和上传机制都不同，所以这里保存和上传的方法都是半空状态，也就是说数据已经准备好了，还缺少具体的保存上传操作，使用时补全就OK。
（建议不同的模块日志分类保存。如业务层和网络层的数据分文件保存）


最后 WLog.java
![这里写图片描述](http://img.blog.csdn.net/20151020224824375)

WLog 类似系统的Log类，是一系列的重载调用方法。我们摘一小段来看：

	/*特别说明的
	* isUpLoad ：当isUpLoad为true时，说明这条日志是要忽视之前设置的上传保存级别，直接上传保存的。（只要总开关还开着)
	*/
	public static void v(String tagString , String explainString ,Throwable throwable, boolean isUpLoad){
        println(LogConfig.VERBOSE, tagString, explainString,throwable,isUpLoad);
    }

    public static void v(String tagString , String explainString){
        println(LogConfig.VERBOSE, tagString, explainString,false);
    }

    public static void v(String tagString , String explainString , boolean isUpLoad){
        println(LogConfig.VERBOSE, tagString, explainString,isUpLoad);
    }

    public static void v(String tagString , String explainString,Throwable throwable){
        println(LogConfig.VERBOSE, tagString, explainString,throwable,false);
    }
