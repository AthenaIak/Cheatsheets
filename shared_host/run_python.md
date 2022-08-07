Disclaimer: This is something that I did about 2 years ago, so I don't remember details about why I took certain decisions or all the steps I followed.

# 1. Install python

I think that the shared host had Python 2 installed but I wanted to use Python 3 instead.
They refused to update, so I looked around for a solution.
Most ways of installing python required permissions, with the exception of miniconda.
I assume that the difference is that other software get installed for the whole machine, as opposed to just the current user.
I followed the instruction to install miniconda on Linux and now python3.7 became available.
Moreover, venv was also included, and this allows you to install any python package you want without permissions (since the packages are installed in the local virtual environment and not centrally in the machine).

So, install minicoda for your user.
You should be able to run python on the server (e.g. create a script that you can run by logging into the server).

# 2. Create a python application that accepts requests to http endpoints

Create a folder where you will host your application's source code.
In my case, I wanted to use python flask, so I named it "hello_flask".
```bash
mkdir public_html/hello_flask
vi hello.py
```

In hello.py, I put the following code to print "Hello, World!" when the "/hello" endpoint is accessed.
```python
from flask import Flask

app = Flask(__name__)

@app.route('/hello')
def index():
    output = 'Hello, World!'
    return output

```

# 3. Install an HTTP server

To execute scripts or applications from a client, you need to use an HTTP server which will redirect external requests to specific processes.
For this, I used Gunicorn.

Go to the application's folder and use miniconda to create a new venv. 
Then, install in the venv gunicorn and flask.

Check how the directory looks after installing tha packages and running the application (next step):
```
$ ls -l ~/public_html/hello_flask/
total 24
drwxrwxr-x  5 texorcc1 texorcc1 4096 Aug  7 03:03 ./
drwxr-x--- 11 texorcc1 nobody   4096 Aug  7 01:51 ../
drwxrwxr-x  2 texorcc1 texorcc1 4096 Sep 27  2020 __pycache__/
-rw-rw-r--  1 texorcc1 texorcc1  130 Sep 27  2020 hello.py
drwxrwxr-x  2 texorcc1 texorcc1 4096 Aug  7 02:11 logs/
drwxrwxr-x  5 texorcc1 texorcc1 4096 Aug  7 02:57 venv/
```
Note that the `logs` directory is created by Gunicorn, so don't create this.
If you need to store logs for your application, choose a different directory name or location.
You will find server and access logs there in case you want to investigate issues.
You should also keep in mind that these logs accumulate and can result in large files.
You can manually clean them up periodically, or create a more automated solution in the future.


# 4. Run the application

To run the application, you can do:
```bash
# find the certificate and key, so that the application runs with https and looks secure from the client perspective
# Note: replace {{user}} and {{domain_name}} bellow with what is valid for your website
certfile=$(ls -t ~/ssl/certs/*crt | head -1)
keyfile=$(ls ~/ssl/keys/$(sed 's;^/home/{{user}}/ssl/certs/{{domain_name}}_com_\(\([0-9a-f]*_\)\{2\}\).*.crt;\1;g' <<< $certfile)*)

cd ~/public_html/hello_flask
source venv/bin/activate
gunicorn -w 3 -b 0.0.0.0:8080 --access-logfile logs/access.log --log-file logs/server.log --certfile=$certfile --keyfile=$keyfile hello:app &
```

The application will listen to the 8080 port and the HTTPS protocol will be used for communication.
Instead of 8080, you can use any port you want, as long as no other application is using it.
Now, you should be able to get the "Hello, World!" response by going to your browser and accessing:
```
https://{{domain_name}}.com:8080/hello
```


# 5. Starting the application automatically

That's all good, but what if the server is restarted or your application stops due to an temporary environmental issue or because it encountered an unexpected error?

I created a script so that I can start/stop the application more easily.

```bash
mkdir -p ~/release/scripts
vi ~/release/scripts/run_process.sh
```

I hadcoded some important values. We can create the same script and set these values as variables/constants instead.
However, we will need to change some regexes for this to work.
So, unless you want to take on this task, simply replace the {{user}} and {{domain_name}} placeholder bellow with what is valid for your website

```bash
#!/bin/bash

if [ $# -ne 2 ]; then
    echo "You must provide two arguments: application_name start/stop"
    exit
fi

app_name=$1
action=$2

function setup_ssl_vars {
    certfile=$(ls -t /home/{{user}}/ssl/certs/*crt | head -1)
    keyfile=$(ls /home/{{user}}/ssl/keys/$(sed 's;^/home/{{user}}/ssl/certs/{{domain_name}}_com_\(\([0-9a-f]*_\)\{2\}\).*.crt;\1;g' <<< $certfile)*)
}


if [[ $app_name = hello_flask ]]; then
    processes="$(ps -ef | grep {{user}} | grep hello_flask | grep -v 'grep hello_flask' | grep -v 'run_process.sh hello_flask')"
    if [[ "$processes" == "" ]]; then
        num_processes=0
    else
        num_processes=$(echo "$processes" | wc -l)
    fi
    master_pid=$(echo "$processes" | awk '{if ($3 == 1) {print $2}}')

    if [[ $action == start ]]; then
        if [[ $num_processes > 0 ]]; then
            echo "Server is already running with pid=$master_pid";
        else
            cd /home/{{user}}/public_html/hello_flask
            source venv/bin/activate
            setup_ssl_vars
            echo "Will run: gunicorn -w 3 -b 0.0.0.0:8080 --access-logfile logs/access.log --log-file logs/server.log --certfile=$certfile --keyfile=$keyfile hello:app &"
            gunicorn -w 3 -b 0.0.0.0:8080 --access-logfile logs/access.log --log-file logs/server.log --certfile=$certfile --keyfile=$keyfile hello:app &
            echo "Server started"
        fi
    elif [[ $action == stop ]]; then
        if [[ $master_pid != "" ]]; then
            echo "Stopping server with pid=$master_pid"
            kill $master_pid
        else
            echo "Server not running"
        fi
    fi
fi
```

With this script, you can run just 1 command to start the application:
```bash
cd
./release/scripts/run_process.sh hello_flask start
```

You can now set up cron jobs to run this script and start the application. 
The script checks if the application is already running, and does nothing if that's the case.
So, it's ok to try starting the application even if it's already running.

I set 2 crojobs (using cpanel, in my case):
1. Every 30 minutes run the command: `/home/{{user}}/release/scripts/run_process.sh hello_flask start`
2. Every day at 00:01 the command: `/home/{{user}}/release/scripts/run_process.sh hello_flask stop;/home/{{user}}/release/scripts/run_process.sh hello_flask start`

I don't care too much if the application is unavailable for a few minutes, so I was ok with just starting the application every 30 minutes if it is stopped.
I also thought that it would be a good idea to restart the application once a day, in case it doesn't do perfect garbage collection, or breaks for some other reason.

# 5. Next steps

There are few things that need to be done in order to manage a site like this more easily:
* Logs management: Create a log folder for the application. Create cron jobs both for the application and the Gunicorn logs. Split the logs to multiple files. Compress older log files and delete them after a predefined duration (maybe upload them somewhere externally if we need to keep them for longer durations).
* Create a deployment process. We shouldn't be changing the python files by hand. We could put them in a dedicated folder that is replaced every time we release a new version.

In any case, this solution is only for small sites and fun projects.
It's a shared host, after all, and we have very limited resources.
