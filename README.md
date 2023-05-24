### Forked Firebase Android Open Source Development with Firebase-DB adjustment to route Websocket over Proxy Socket connection

#### Origin Readme: https://github.com/firebase/firebase-android-sdk#firebase-android-open-source-development

#### Example with Socks proxy including authentication

1. Assemble jar ```gradle firebase-database:assembleRelease```
2. Optional: set socksProxyUsername & socksProxyPassword in the lcoal.properties
3. Optional: Read properties in build.gradle


```
android {

    def properties = new Properties()
    File localPropertiesFile = rootProject.file("local.properties")
    if (localPropertiesFile.exists()) {
        properties.load(new FileInputStream(localPropertiesFile))
    }
   
    ...
    
     buildTypes.each {
        it.buildConfigField("String", "SocksProxyUsername", "\"${properties['socksProxyUsername']}\"")
    
    ...

```
3. Create tunnel socket witch apache SSlSocketFactory


```
private const val RFC_1928_SOCKS_PORT = 1080
private const val HTTPS_PORT = 443

@Suppress("DEPRECATION")
@kotlin.jvm.Throws(IOException::class)
fun getTunnelSSLSocket(dbUrl: String): Socket? {
    val (socksAuthUser, socksAuthPasswd) = getSocksAuth() ?: return null
    val (host, httpsPort) = Pair(dbUrl, HTTPS_PORT)
    val proxyHost = System.getProperty("http.proxyHost") ?: return null
    val underlyingSocket =
        Socket(Proxy(Proxy.Type.SOCKS, InetSocketAddress(proxyHost, RFC_1928_SOCKS_PORT)))
    Authenticator.setDefault(getAuth(socksAuthUser, socksAuthPasswd))
    underlyingSocket.tcpNoDelay = true
    underlyingSocket.keepAlive = true
    underlyingSocket.reuseAddress = true
    underlyingSocket.connect(InetSocketAddress(host, httpsPort))
    return SSLSocketFactory.getSocketFactory()
        .createSocket(underlyingSocket, host, httpsPort, true)
}

private fun getSocksAuth(): Pair<String, CharArray>? {
    return if (BuildConfig.SocksProxyUsername == "null" || BuildConfig.SocksProxyPassword == "null") {
        null
    } else {
        Pair(BuildConfig.SocksProxyUsername, BuildConfig.SocksProxyPassword.toCharArray())
    }
}

private fun getAuth(user: String, password: CharArray): Authenticator {
    return object : Authenticator() {
        public override fun getPasswordAuthentication(): PasswordAuthentication {
            return PasswordAuthentication(user, password)
        }
    }
}
```
4. Get firebase db instance with proxySocket

```
 FirebaseDatabase.getInstance(firebaseInstanceDbUrl, socket)
```

Thats it ;)