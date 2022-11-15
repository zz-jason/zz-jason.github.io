---
title: "[MySQL] Build, test and debug MySQL server"
date: 2022-11-15T02:59:46Z
categories: ["MySQL"]
draft: true
---

## Build from source

* MySQL version: mysql-8.0.31
* commit hash: a246bad76b9271cb4333634e954040a970222e0a



Build from source:

```sh
cmake -S . -B build \
      -DCMAKE_BUILD_TYPE=Debug \
      -DDOWNLOAD_BOOST=1 \
      -DWITH_BOOST=. \
      -DCMAKE_INSTALL_PREFIX=`pwd`/mysql-debug-install \
      -G=Ninja

cmake --build build -j `nproc`
```



Create `my.cnf`:

```
[mysqld]
datadir=/tmp/mysql/data
```



Init the config and data dir (only required in the first run):

```sh
mkdir /tmp/mysql/data

build/bin/mysqld --defaults-file=my.cnf \
                 --initialize-insecure \
                 --user=`whoami`
```



## Start MySQL

```sh
build/bin/mysqld --defaults-file=my.cnf
```



Connect and verify

```sh
build/bin/mysql -h 127.0.0.1 -P 3306 -u root -e "select version()"
```



## Debug

To debug in VS Code, create `launch.json` firstly:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
                "name": "Attach & Debug",
                "type": "cppdbg",
                "request": "attach",
                "program": "${workspaceFolder}/build/bin/mysqld",
                "linux": {
                    "name": "Attach & Debug",
                    "type": "cppdbg",
                    "request": "attach",
                    "program": "${workspaceFolder}/build/bin/mysqld",
                }
            }
        ]
}
```



> If you met **"ptrace: Operation not permitted"** error, add a file called `10-ptrace.conf` to `/etc/sysctl.d/` and add the following `kernel.yama.ptrace_scope = 0`. See [Debug C++ in Visual Studio Code](https://code.visualstudio.com/docs/cpp/cpp-debug) for details.

