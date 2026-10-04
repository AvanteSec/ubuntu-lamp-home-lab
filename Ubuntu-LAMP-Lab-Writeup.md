# Ubuntu LAMP Home Lab — Progress Write-up
**Author:** Avante Williams  
**Lab sessions:** October 3–4, 2026  
**Status:** Core LAMP lab completed; optional extensions remain

## Objective
Build a Linux web server in a VirtualBox virtual machine, test its components, and document installation, troubleshooting, and verification. This is a foundation for further work with web applications, server administration, and security.

LAMP stands for Linux, Apache, MySQL, and PHP. Linux runs the system, Apache serves web requests, MySQL stores data, and PHP runs application code on the server.

## Environment
| Component | Configuration |
|---|---|
| Host | Windows laptop |
| Virtualization | Oracle VirtualBox 7.2.14 |
| VM | UBUNTU-LAMP |
| Guest hostname | Ubuntu-LAMP |
| Ubuntu account | avante |
| Guest OS | Ubuntu 26.04.1 LTS, Resolute Raccoon |
| Assigned resources | 4 GB RAM, 4 CPUs |
| Virtual disk | 100 GB |
| Display | VMSVGA, 128 MB video memory, 3D acceleration disabled |
| Timezone | America/Chicago |

## Working method and evidence
I performed the installation and commands on my own computer. I used ChatGPT/Codex for explanations and troubleshooting guidance, sharing screenshots of installer screens, terminal output, errors, and browser results.

My process was to share the observed result, interpret it with assistance, take the next action, and share follow-up evidence. AI assistance did not operate my VM directly. The screenshots support the results described below; they do not establish every possible underlying cause.

This draft records the screenshot evidence but does not embed the original images. Screenshots containing passwords must be excluded or redacted before publication.

## 1. Ubuntu installation
I installed Ubuntu in the virtual machine using the interactive installer and default application selection. I used the virtual wired connection, did not select third-party graphics or media software, and chose no disk encryption for this practice VM.

The erase-disk option applied to the VM's virtual disk. I created my Ubuntu account, required a password for login, and selected America/Chicago. I skipped Ubuntu Pro and left location services off.

After installation, I verified the operating system:
```bash
cat /etc/os-release
```
The screenshot showed Ubuntu 26.04.1 LTS and the codename resolute.

## 2. System updates
I refreshed the package list:
```bash
sudo apt update
```
The output reported 57 packages available for upgrade. I then upgraded the installed software:
```bash
sudo apt upgrade
```
The upgrade completed and returned to the terminal prompt. I restarted through the desktop after encountering a reboot inhibitor, described below.

## 3. Apache web server
I installed Apache:
```bash
sudo apt install apache2
```
I opened Firefox inside Ubuntu and visited:
```text
http://localhost
```
The Apache2 default page displayed “It works!” My browser screenshot confirmed Apache was serving a page inside the VM. Here, localhost refers to the Ubuntu VM because the browser was running inside it.

## 4. MySQL database server
I installed MySQL:
```bash
sudo apt install mysql-server
```
I checked its service:
```bash
systemctl status mysql --no-pager
```
The screenshot showed active (running), an enabled service, and “Server is operational.” The MySQL client later reported version 8.4.11.

## 5. PHP installation and browser test
I installed PHP and the packages for Apache and MySQL integration:
```bash
sudo apt install php libapache2-mod-php php-mysql php-cli
sudo systemctl restart apache2
php -v
```
The terminal reported PHP 8.5.4. I created a test file:
```bash
sudo nano /var/www/html/test.php
```
Its contents were:
```php
<?php
echo "PHP is working!";
?>
```
After saving it, I opened http://localhost/test.php in Firefox. The page displayed “PHP is working!” This confirmed Apache could execute PHP and return its output to the browser.

## 6. Database and application account
I opened MySQL with administrative access:
```bash
sudo mysql
```
I created and verified my practice database:
```sql
CREATE DATABASE lamp_lab;
SHOW DATABASES;
```
The result included lamp_lab.

I created a separate application account and limited its data permissions to that database:
```sql
CREATE USER 'lamp_user'@'localhost' IDENTIFIED BY '<private-password>';
GRANT SELECT, INSERT, UPDATE, DELETE
ON lamp_lab.* TO 'lamp_user'@'localhost';
```
The password above is a documentation placeholder, not a value to copy.

I tested login from the Ubuntu terminal:
```bash
mysql -u lamp_user -p lamp_lab
```
After logging in, I checked the account:
```sql
SHOW GRANTS FOR CURRENT_USER;
```
The screenshot showed SELECT, INSERT, UPDATE, and DELETE on lamp_lab.* and a global USAGE entry. The account was not granted CREATE TABLE or global administrative privileges.

During setup, I initially entered a sample password placeholder literally, then changed the account password. Actual passwords are omitted from this record. Some original screenshots reveal credentials; these are unsuitable for publication without redaction, and the exposed password should be replaced before further use.

## 7. Table creation and data test
Using the administrative MySQL session for lamp_lab, I created a table:
```sql
CREATE TABLE messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(255) NOT NULL
);
```
The id field assigns a unique number automatically. The message field stores text up to 255 characters and cannot be NULL.

I inserted a message and retrieved it:
```sql
INSERT INTO messages (message)
VALUES ('My LAMP lab is working!');

SELECT * FROM messages;
```
The screenshot showed:
| id | message |
|---|---|
| 1 | My LAMP lab is working! |

This confirmed that MySQL saved and returned the data. It did not yet verify PHP connecting to MySQL.

## 8. Restoring and checking the checkpoint
On October 4, I reported restoring the VM from a snapshot. Before continuing, I checked the services and stored data:
```bash
systemctl is-active apache2 mysql
sudo mysql lamp_lab -e "SELECT * FROM messages;"
```
Both services returned active, and the database returned the original message. I shared a screenshot confirming these results. The exact restored snapshot name was not recorded.

## 9. Connecting PHP to MySQL
I created a private PHP configuration outside Apache's website directory:
```bash
sudo nano /etc/lamp-lab.php
```
The configuration returns the connection details:
```php
<?php
return [
    'host' => 'localhost',
    'database' => 'lamp_lab',
    'username' => 'lamp_user',
    'password' => '<private-password>'
];
```
The password shown here is only a documentation placeholder. I reported completing the file and these permissions commands; the terminal screenshot also showed that the commands were entered:
```bash
sudo chown root:www-data /etc/lamp-lab.php
sudo chmod 640 /etc/lamp-lab.php
```
This assigns ownership to root and the www-data group, allows the owner to read/write and the group to read, and gives other users no access. A separate file-permission listing was not captured.

I then created the database test page:
```bash
sudo nano /var/www/html/db-test.php
```
The supplied page code was:
```php
<?php
$config = require '/etc/lamp-lab.php';

try {
    $db = new PDO(
        "mysql:host={$config['host']};dbname={$config['database']};charset=utf8mb4",
        $config['username'],
        $config['password'],
        [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
    );

    echo "<h1>My LAMP Lab</h1>";

    $results = $db->query("SELECT message FROM messages ORDER BY id");

    foreach ($results as $row) {
        echo "<p>" . htmlspecialchars(
            $row['message'], ENT_QUOTES, 'UTF-8'
        ) . "</p>";
    }
} catch (PDOException $e) {
    http_response_code(500);
    error_log($e->getMessage());
    echo "Database connection or query failed.";
}
```
The page loads the separate configuration, connects through PDO, reads the messages table, and escapes message text before displaying it as HTML. It logs database exceptions and shows a generic failure message to the browser.

After troubleshooting the login failure described below, I opened http://localhost/db-test.php in Firefox. My screenshot showed the heading “My LAMP Lab” and the message “My LAMP lab is working!”

**Verified result:** The browser requested the page from Apache, Apache executed PHP, PHP retrieved the stored message from MySQL, and the response displayed it in the browser. The core LAMP integration test was complete.

## Troubleshooting record

### Initial VM startup and display issues
**Observation:** The VM initially had blank or slow startup behavior. Diagnostic information included a vmwgfx warning, Windows hypervisor activity, and a VirtualBox “Snail execution mode” message.

**Investigation:** I shared screenshots and diagnostic information for guidance. We reviewed VM settings and possible host virtualization interactions. We did not establish a single root cause.

**Outcome:** Following a Windows restart and another normal VM boot, Ubuntu started successfully. Later, I reported that terminal typing was not laggy.

**Limit:** The successful boot does not prove that any particular diagnostic message or setting caused the original problem.

### Installer crash-report notification
**Observation:** During installation, a “System program problem detected” notification appeared.

**Action:** I shared the screenshot and dismissed the notification, allowing installation to continue.

**Verification:** Installation completed, and I reached the Ubuntu desktop. The specific program behind the report was not identified.

### Reboot blocked after updates
**Observation:** Running sudo reboot produced an “Operation inhibited” message naming the GNOME session and a logged-in user.

**Action:** I shared the terminal screenshot and selected the desktop's “Restart & Install” option.

**Verification:** The restart worked and I continued using Ubuntu afterward.

### Web address entered as a terminal command
**Observation:** I typed http://localhost/test.php into Bash. It returned “No such file or directory.”

**Investigation:** My screenshot showed that the URL had been entered at the shell prompt.

**Action:** I opened the address in Firefox inside Ubuntu.

**Verification:** A follow-up screenshot displayed “PHP is working!”

**Lesson:** A browser address should be opened in a browser; entering it alone in Bash makes the shell attempt to execute it.

### Ubuntu and database password confusion
**Observation:** sudo mysql lamp_lab prompted for a password, and several attempts failed.

**Investigation:** The screenshot showed that the failing prompt came from sudo, before MySQL opened.

**Action:** I used my Ubuntu account password for sudo. This is separate from the lamp_user database password.

**Verification:** My next screenshot showed the MySQL welcome message and mysql> prompt. I then created the messages table successfully.

### Pasting PHP code into the VM
**Observation:** I could not paste the supplied PHP code into the Ubuntu terminal/nano editor.

**Guidance and action:** Guidance first covered Ctrl+Shift+V and VirtualBox's shared clipboard. When pasting still failed, I was guided to open this conversation in Firefox inside Ubuntu, copy the code there, and paste it into nano within the same guest system.

**Verification and limit:** The later browser screenshot showed that the PHP page had been created and executed. I did not separately confirm the exact paste method that succeeded or whether Guest Additions/shared clipboard were configured.

### PHP-to-MySQL login failure
**Observation:** The first browser test displayed “Database connection or query failed.” I shared a screenshot of the page rather than assuming the installation had failed.

**Investigation:** With guidance, I inspected the Apache error log:
```bash
sudo tail -n 20 /var/log/apache2/error.log
```
I shared the terminal output. Its relevant entry showed SQLSTATE[HY000] [1045] Access denied, with a password being used. This pointed to a rejected database login. The log also contained Apache ServerName warnings, but these did not explain the MySQL authentication error.

**Corrective guidance:** I was instructed to test the application account directly:
```bash
mysql -u lamp_user -p lamp_lab
```
If that login worked, the next step was to check that /etc/lamp-lab.php used lamp_user and the same working database password. A password reset was offered if the direct login failed.

**Outcome:** I reported “Got it” and shared a refreshed browser screenshot displaying the database message successfully. This confirms the connection issue was resolved. I did not explicitly state which credential field I corrected or whether I reset the password, so this record does not claim a specific change.

**Lesson:** The generic browser error indicated a failure; the server log supplied the useful diagnostic detail. Checking authentication separately helps distinguish credentials from service or application problems.

## Screenshot evidence recorded
| Evidence | What it supports |
|---|---|
| Ubuntu installer screens and desktop | Installation choices and a usable desktop |
| /etc/os-release output | Guest OS identification |
| apt update and completed upgrade output | Package refresh and upgrade progress |
| Reboot inhibitor output | Reason the terminal reboot was blocked |
| Apache default page | Browser-to-Apache test |
| MySQL service status | Running database service |
| PHP version output | Installed command-line PHP version |
| Bash URL error followed by PHP browser page | Error, correction, and successful Apache/PHP test |
| SHOW DATABASES output | Creation of lamp_lab |
| Application login and SHOW GRANTS output | Login and database permissions |
| Failed sudo attempts followed by MySQL prompt | Authentication troubleshooting and recovery |
| CREATE TABLE, INSERT, and SELECT output | Table creation and successful data storage/retrieval |
| Restored VM service and SELECT checks | Services and data available after restoration |
| Generic database failure in Firefox | Initial PHP-to-MySQL test failure |
| Apache error log with MySQL error 1045 | Rejected database login |
| Successful db-test.php browser page | Complete LAMP integration test |

## Snapshot and stopping point
I reported taking an earlier baseline snapshot and restoring a snapshot on October 4. The service and data checks confirmed that the restored state included the previous database work.

After the successful PHP database test, a new checkpoint snapshot was recommended with the name “LAMP — PHP database connection working.” Creation of this newest snapshot has not been confirmed. Starting the VM normally preserves ongoing changes; restoring a snapshot returns it to the saved checkpoint.

## Current status
| Check | Status |
|---|---|
| Ubuntu installed and usable | Confirmed |
| System package upgrade | Completed |
| Apache serves a webpage | Confirmed |
| Apache executes PHP | Confirmed |
| MySQL service runs | Confirmed |
| Application account login and grants | Confirmed during initial setup |
| Database stores and retrieves a message | Confirmed |
| Private configuration and permissions commands | Completed as reported; commands visible in screenshot |
| PHP connects to MySQL | Confirmed by successful page |
| Full browser-to-database application test | Confirmed |
| Snapshot of final working state | Recommended; not yet confirmed |

**Core project result:** I built and verified a working LAMP lab and used screenshots and server logs to troubleshoot installation, command usage, authentication, and application connectivity.

This is a local practice environment. No complete server-hardening or security assessment has been performed. Earlier exposed credentials should be replaced, and screenshots containing them must be omitted or redacted before sharing publicly. The screenshots are catalogued as evidence; original images are not embedded in this Markdown file.

## Optional future work
A message-entry form could extend the page to insert new data through prepared SQL statements. Further exercises could cover access and error logs, server configuration, backups, and security controls. These are optional follow-up projects, not requirements for completing this core setup.

## Skills practiced
Virtual machine setup, Linux package management, service verification, terminal and browser use, basic PHP, database creation, SQL queries, limited application-account permissions, PDO database connectivity, private configuration files, HTML output escaping, error-log investigation, snapshot verification, and screenshot-based troubleshooting.

