# Setup a web server


**==This article is copied from https://www.git-tower.com/blog/apache-on-macos/, tested on Mac OS Monterey Version 12.0.1 (21A559), works.==** 


***

At [Tower](https://www.git-tower.com/), we use [Apache](https://httpd.apache.org/) to host websites like our [learning platform](https://www.git-tower.com/learn), our [blog](https://www.git-tower.com/blog), and our [main product site](https://www.git-tower.com/). While this doesn’t mean we have to use Apache to run the sites locally, using a similar stack for development and production is generally a good idea. This means setting up a development environment with Apache. On macOS, which is what we use for the most part, there are quite a few options. We could set up Apache in a virtual machine, perhaps controlled using [Vagrant](https://www.vagrantup.com/). We could use [Docker](https://www.docker.com/) to run Apache in a container. There are also solutions with graphical user interfaces like [MAMP](https://www.mamp.info/en/mac/). However, a convenient and simple solution is to just set up Apache running natively in macOS — no wrappers, no virtual machines, no containerization. In this post, we’ll go through how to set up Apache and PHP, using versions installed using the [Homebrew package manager](https://brew.sh/) for macOS.

macOS comes with built-in versions of Apache and PHP, and we could easily use those. However, there are a few drawbacks with this approach. We don’t have control over the exact versions used, and the version available might not be up to date. I’ve had problems where OS updates have overwritten my configuration for the built-in Apache server. Finally, running `php -v` to check the version of the built-in PHP gives a warning message stating that PHP will be removed from future versions of macOS — in fact, in the upcoming macOS Monterey, PHP [seems to be gone](https://mjtsai.com/blog/2021/06/16/macos-12-removes-php/). Instead of using the built-in versions, we’ll install Apache and PHP using Homebrew. We'll look at how to set up a local development environment using these and, as usual, we’ll try to cover the “why” as well as the “what”; in addition to just presenting the configuration, we'll go over the purpose of each directive and command.

Here are the steps we'll take:


1. [Installing Apache and PHP](https://www.git-tower.com/blog/apache-on-macos/#installation)
2. [Configuring Development URLs](https://www.git-tower.com/blog/apache-on-macos/#development-urls)
3. [Apache Configuration](https://www.git-tower.com/blog/apache-on-macos/#apache-configuration)
4. [PHP Configuration](https://www.git-tower.com/blog/apache-on-macos/#php-configuration)
5. [Virtual Host Setup](https://www.git-tower.com/blog/apache-on-macos/#virtual-host-setup)
6. [Up and Running](https://www.git-tower.com/blog/apache-on-macos/#up-and-running)

## 1. Installation

Instructions for how to install Homebrew itself can be found on [the official Homebrew website](https://brew.sh/). Assuming Homebrew is installed, all you need to do in order to install Apache and PHP is to run the following command: `brew install httpd php` (`httpd` refers to the Apache web server).

A word regarding paths: on a Mac with Apple Silicon, Homebrew will use `/opt/homebrew` as its prefix. The prefix is sort of a base directory; a directory under which Homebrew will put various files belonging to the packages it installs. Binary files will go in `/opt/homebrew/bin`, configuration files in `/opt/homebrew/etc/`, and so on. On an Intel-based Mac, this base directory will likely be `/usr/local` instead. In this article, I’ll refer to paths using a Homebrew prefix of `/opt/homebrew`. If your prefix is different, please change the paths accordingly. To find out which base directory Homebrew is using on your machine, run `brew --prefix`.

## 2. Development URLs

Before we get started on Apache and PHP configuration, let’s touch on the topic of development URLs briefly. I think a nice setup for local development is to use a specific TLD like `.test`, accessing my various projects through URLs like `my-first-project.test`, `my-second-project.test`, and so on. In this guide we’ll simply use the `/etc/hosts` file to point our “fake” domains at our local web server. If there’s interest, a future post might cover how to set up a service like [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) to automatically grab all requests to hosts ending with `.test` and send them to our local development server.

`.test` is a reserved top-level domain and so, using this, you shouldn’t run into problems such as the people using `.dev` for development environments did when [Google acquired the ](https://medium.engineering/use-a-dev-domain-not-anymore-95219778e6fd)`.dev` TLD.

The `/etc/hosts` file provides a convenient way to associate host names with IP addresses. Normally, when we visit a URL like `https://www.git-tower.com/`, the actual IP address of the server has to be determined through something called the Domain Name System, or DNS for short. The `/etc/hosts` file gives us an easy way to override this. For the purposes of this article, let’s say we’re working with a project called `my-project`, which we want served at the URL `my-project.test`. We’ll add this line at the bottom of `/etc/hosts`:

```
127.0.0.1 my-project.test
```

After saving the file, visiting `my-project.test` in a browser will result in the request being sent to the IP address for our own machine.

## 3. Apache Configuration

Next, let’s get to work on the actual Apache configuration. In my case, the main Apache configuration file is located at `/opt/homebrew/etc/httpd/httpd.conf` (again, on an Intel-based Mac, this is likely to be `/usr/local/etc/httpd/httpd.conf`). In this file, there are a few changes to make:

```
Listen 8080
```

This line tells Apache to listen for traffic on the port 8080. Accessing ports with numbers lower than 1024 require root privileges and so, listening on port 8080 lets users run Apache without being root. However, as HTTP traffic goes to port 80 by default, we want to listen on that port instead:

```
Listen 80
```

Chances are, you want to run multiple websites on your computer, with several hostnames in `/etc/hosts`. However, all of these will hit the same Apache server. To serve multiple sites from one Apache server, Apache can look at the hostname of the incoming request and pass the request to one of multiple *virtual hosts*. Virtual host support has to be enabled by removing the `#` in front of the line below, turning it from a comment into an Apache directive that loads the virtual hosts configuration from the file in question:

```
#Include /opt/homebrew/etc/httpd/extra/httpd-vhosts.conf
```

We’ll uncomment another line in order to load the `mod_rewrite` module. This module is used to rewrite incoming URLs. For example, many web frameworks use it to enable “pretty URLs”, letting site visitors use URLs like `/posts/2021/some-post-title/` while translating them into URLs like `/index.php?p=697` for the back-end. This module will likely be useful, so let’s enable it by removing the `#` below:

```
#LoadModule rewrite_module lib/httpd/modules/mod_rewrite.so
```

Apache comes with configuration for a default site, with a document root of `/opt/homebrew/var/www`. We won’t be using this site, as we’ll use virtual hosts instead. As it stands, everything would work, but Apache would emit a warning every time it started, stating that it can’t determine the hostname to use for this default site. We’ll set the hostname on a per-virtual host basis later. However, to silence the warning message, we’ll uncomment the default `ServerName` directive in `httpd.conf` (or just enter any hostname you want):

```
ServerName www.example.com:8080
```

## 4. PHP Configuration

We want PHP to be available on our server. For this, we’ll add another `LoadModule` directive after the other ones, loading a module provided by PHP as installed through Homebrew earlier (on an Intel-based Mac, change `/opt/homebrew` to `/usr/local`):

```
LoadModule php_module /opt/homebrew/opt/php/lib/httpd/modules/libphp.so
```

This loads the PHP module. There is some additional configuration required. Let’s add further PHP configuration in a separate file in the `extra` directory, just like with the virtual hosts configuration. We need to include it explicitly from the main configuration file, so find the section with `Include` directives and add this after the last one:

```
# PHP settings
Include /opt/homebrew/etc/httpd/extra/httpd-php.conf
```

The rest of our PHP configuration goes in the file `/opt/homebrew/etc/httpd/extra/httpd-php.conf`:

```
<IfModule php_module>
  <FilesMatch .php$>
    SetHandler application/x-httpd-php
  </FilesMatch>

  <IfModule dir_module>
    DirectoryIndex index.html index.php
  </IfModule>
</IfModule>
```

We start by checking if the PHP module is available, which might seem redundant as we just added it in the `httpd.conf` file. However, these files may get edited independently of each other, so let’s follow the convention of the other files in the `extra` directory and check that the modules we use are available. We’ll look for files ending in `.php`, and set their handler to `application/x-httpd-php` — a handler provided by the PHP module. A handler in Apache represents an action to take for a file. While most files are just served using a built-in handler, the PHP files have to be interpreted by PHP before being served.

Some guides use the `AddType` or `AddHandler` directives here, which take parameters for the file extensions they apply to. These can [introduce security issues](https://bugs.php.net/bug.php?id=45229) as they check their configured extensions against every extension of a file. For example, a web application may allow users to upload `.jpeg` files. However, if a user uploads a file with a name ending in `.php.jpeg`, this file might be executed as PHP if the above directives are used. Therefore, we use the `FilesMatch` directive along with `SetHandler`. Not that we're likely to encounter these issues when we're running Apache locally for ourselves, but we might as well set things up properly.

The `DirectoryIndex` directive makes sure that if a URL for a directory is requested, and the directory contains an `index.php` file (or an `index.html` file), that file will be served.

## 5. Virtual Host Setup

We’ve now set up Apache to support PHP and virtual hosts. We still need to add configuration for each virtual host separately. Earlier, we uncommented a line in order to include the `/opt/homebrew/etc/httpd/extra/httpd-vhosts.conf` file (`/usr/local/etc/httpd/extra/httpd-vhosts.conf` on an Intel-based Mac). Now, let’s edit it to actually configure a virtual host.

```
<VirtualHost *:80>
    ServerName my-project.test
    DocumentRoot /path/to/my-project

    <Directory /path/to/my-project>
        Require all granted
        AllowOverride All
    </Directory>
</VirtualHost>
```

This configures a virtual host on port 80. `ServerName` sets the hostname for the virtual site. When we visit `my-project.test` in the browser, the change we made to `/etc/hosts` will make sure the request is sent to the local Apache server. Apache will find the virtual host with the corresponding hostname and serve that site. The `DocumentRoot` directive specifies the location on the file system where the files to be served exist. Make sure to change this path, along with the other instance of it in the `Directory` directive, to the actual path of your website on your machine!

The `Directory` directive has to do with permissions. In the `httpd.conf` file, there’s a section which denies access to any resource by default. Access to anything that should be public has to be specifically allowed. So, for the files in our site root, we give everyone access to all resources through the `Require` directive. The `AllowOverride` directive controls which directives can be overridden in a `.htaccess` file. A `.htaccess` file can be used for per-directory configuration — many CMSes use this file together with the `mod_rewrite` module mentioned earlier to set up their URLs, for example. Here, we allow all directives to be overridden.

While we’re at it, we can go ahead and remove or comment out the existing dummy virtual hosts existing in this file, otherwise Apache will emit some warnings on startup, as the document roots for these sites likely don’t exist.

## 6. Up and Running

That’s all the configuration done! All that remains is to start the server. If you want Apache to start automatically with your computer, you can use Homebrews’ `services` command to start the server and enable it for autostart at the same time:

```
sudo brew services start httpd
```

`sudo brew services stop httpd` will then both turn off the server and disable autostart for it.

Personally, I tend to just use Apache’s built-in “server control interface” to start the server as I need it: `sudo apachectl start` will start the server and `sudo apachectl stop` will stop it. Hopefully, after starting the server, visiting your chosen hostname in your browser will result in your site being served correctly!

That’s about all for today! I hope you found this guide useful. Of course, we’ve only covered Apache and PHP — there are many additions you may want to make to this stack, such as the [MySQL database server](https://www.mysql.com/). If you’re interested in a future article covering this or any other aspect of an Apache-based hosting stack, please [let us know](https://twitter.com/gittower)!