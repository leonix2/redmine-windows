# Redmine Install Process

## Configuration:

* Windows Server Edition 2008+
* Redmine Package 4.2.9
* MySQL Server 5.7.39
* Ruby 2.7.5
* Rails 5.2
* Puma 6.0.1
* Apache 2.4


### 1. Download and Install Microsoft Visual C++ 2019 Redistributable Package (x64)
https://aka.ms/vs/17/release/vc_redist.x64.exe

### 2. Install MySQL  server for Windows

- Download the package
> https://dev.mysql.com/get/Downloads/MySQLInstaller/mysql-installer-community-5.7.39.0.msi
-  Start the installer
- Choose Custom installation
- Select products:
    - MySQL Server 5.7.39 - X64
    - MySQL Shell 8.0.30 -X64
- At Configuration stage select "Server Computer" as Config Type, TCP port should be == 3306
- Create Root Password at next screen
- Leave the default values on next two screens
- Press "Execute" on "Apply Configuration" Screen

### 3. Download MySQL Connector for Windows:

> https://cdn.mysql.com//archives/mysql-connector-c/mysql-connector-c-6.1.3-winx64.zip

- Unzip it and copy files from child directory to `c:\mysql-connector` (Create dir before)

### 4. Create MySQL DB and User for Redmine:

- Open MySQL console from Installed programs
- Enter the root password from section 2
- Enter the following commands, change the `<PASSWORD>` to new password which will be used for Redmine DB user

      CREATE DATABASE redmine CHARACTER SET utf8mb4;
      CREATE USER 'redmine'@'localhost' IDENTIFIED BY '<PASSWORD>';
      GRANT ALL PRIVILEGES ON redmine.* TO 'redmine'@'localhost';
- You should see "QUERY OK" after each command. Close MySQL Console.

### 5. Install Apache Web Server
- Download Package
  https://www.apachelounge.com/download/VS17/binaries/httpd-2.4.54-win64-VS17.zip
- Unzip the Apache24 folder to `c:\Apache24` (that is the ServerRoot in the config).
- To install WebServer as a service open command prompt as Administrator and type:
    ```
    cd c:\Apache24\bin
    httpd.exe -k install
    ```
- If you see a warning *"Could not reliably determine the server's fully qualified domain name"* - it's OK, we will set it later in process of installation
- Start Apache Server with the command in prompt:
    ```
    services.msc
    ```
- Select service Apache and press start for it.
- You can test your installation by opening up your Browser and typing in the address:
  http://localhost

### 6. Install Ruby for Windows.

- Download Ruby 2.7.5+devkit package from
> https://github.com/oneclick/rubyinstaller2/releases/download/RubyInstaller-2.7.5-1/rubyinstaller-devkit-2.7.5-1-x64.exe
- start the installer
- In process of installation check  "Add Ruby executables to your PATH" setting.
- Select MSYS2 Development package to install
- After Installing process new console window will be opened, choose 3 - "MSYS2 and MINGW development toolchain", After
  installing process press ENTER to exit.


### 7. Download MIME INFO Package for Debian (Yeah, for Debian :) )

>http://ftp.debian.org/debian/pool/main/s/shared-mime-info/shared-mime-info_2.2-1_amd64.deb

- Unzip the package and .tar file inside it.
- Copy "data" directory to `c:\ruby\mime\` (create dir if needed)
- Open CMD.exe console and set environment variable for mime:
    ```
    setx FREEDESKTOP_MIME_TYPES_PATH C:\Ruby\mime\data\usr\share\mime\packages
    ```
- Close and reopen CMD Window for applying new env variable.

### 8. Import the existent MySQL Database

- Copy DB Dump File to c:\redmine_4-2.sql
- In CMD Console apply the following command's
    ```
    cd "c:\Program Files\MySQL\MySQL Server 5.7\bin"
    mysql.exe -u redmine -p redmine < c:\redmine_4-2.sql
    ```
- Enter Redmine DB User password from section 2.2 for import apply.

### 9. Setting up Redmine and Ruby

- Download Redmine package - https://www.redmine.org/releases/redmine-4.2.9.zip
- Extract it to c:\Webserver\Redmine
- Make Database configuration file by creating file c:\Webserver\Redmine\config\database.yml with following content (You can copy the file database.yml.example and change content of file)
    ```
    production:
    adapter: mysql2
    database: redmine
    host: localhost
    username: redmine
    password: <PASSWORD>
    ```
- In CMD Console apply the following commands one by one:
    ```
      cd c:\Webserver\Redmine
      gem install bundler
      gem install mysql2 --platform=ruby -- '--with-mysql-lib="C:\mysql-connector\lib" --with-mysql-include="C:
      \mysql-connector\include" --with-mysql-dir="C:\mysql-connector"'
      gem install rails:5.2
      gem install puma:6.0.1
      bundle install --without development test rmagick   
      bundle exec rake generate_secret_token
      set RAILS_ENV=production
      bundle exec rake db:migrate
    ```

### 10. Redmine Configuration

Redmine settings are defined in a file named `config/configuration.yml`.

If you need to override default application settings, simply copy `config/configuration.yml.example` to
`config/configuration.yml` and edit the new file; the file is well commented by itself, so you should have a look at it.

Set a path where Redmine attachments will be stored which is different from the default `c:\Webserver\redmine\files` directory of your Redmine instance using the `attachments_storage_path` setting.

Example:
```
attachments_storage_path: D:/redmine/files
```
### 11. Setting up Application and Web Servers for Redmine

#### 11.1 Puma application server.

To have Puma server started automatically with Windows, perform the following steps:

- Create a file, pumastart.bat, in C:\ruby with the following contents:
    ```
    cd C:\redmine
    start /min puma -e production -p 3000 -t 8:32
    ```
- Go to Task Scheduler (Run `taskschd.exe`) | Create a Task.
- Check the Run whether user is logged or not, Run with the highest privileges, and Hidden checkboxes.
- In Actions, go to New | Start a program and find pumastart.bat. (C:\ruby\pumastart.bat)
- On the Triggers tab, click New and choose Begin the task: At startup (located in the top dropdown).

#### 11.2 Apache Web server.

- Open file c:\apache24\conf\httpd.conf
- Uncomment following module lines:
    ```
    LoadModule proxy_module modules/mod_proxy.so
    LoadModule proxy_http_module modules/mod_proxy_http.so
    LoadModule proxy_http2_module modules/mod_proxy_http2.so
    ```
- Set Server name as domain which should be available for external Redmine users
  Example:

    ```
    ServerName redmine.myserver.com:80
    ```
- Create VirtualHost section for your Redmine domain, keep port 3000 unchanged cause it used for Puma application server
    ```
    <VirtualHost *:80>
        ServerName redmine.myserver.com
        ServerAdmin webmaster@myserver.com
        ErrorLog logs/redmine_error_log
        DocumentRoot c:\Webserver\redmine\public
        ProxyVia on
        RewriteEngine on
        ProxyPass / http://redmine.myserver.com:3000/ nocanon
        ProxyPassReverse / http://redmine.myserver.com:3000/
    </VirtualHost>
    ```
- Restart Apache Server via cmd command:
    ```
    c:\Apache24\bin\httpd.exe -k restart
    ```
- If command executed with no errors, in Web browser at address `http://myserver.com` you should see the Redmine web service.

