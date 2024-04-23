# 15 Second Password Hack, Mr. Robot Style (Windows)

This project demonstrates a rapid and stealthy technique to exfiltrate passwords from an unattended Windows PC using a USB Rubber Ducky, inspired by a scene from the hacker show Mr. Robot. It leverages a staged PowerShell downloader to execute Invoke-Mimikatz directly in memory, while avoiding detection by not writing to the hard disk. The payload automates the process to open an admin command prompt, bypass User Account Control (UAC), download, and execute Invoke-Mimikatz, and then upload the captured credentials to a server, all within approximately 15 seconds.

## Disclaimer

The tools and scripts provided in this repository are made available for educational purposes only and are intended to be used for testing and protecting systems with the consent of the owners. The author does not take any responsibility for the misuse of these tools. It is the end user's responsibility to obey all applicable local, state, national, and international laws. The developers assume no liability and are not responsible for any misuse or damage caused by this program. Under no circumstances should this tool be used for malicious purposes. The author of this tool advocates for the responsible and ethical use of security tools. Please use this tool responsibly and ethically, ensuring that you have proper authorization before engaging any system with the techniques demonstrated by this project.

## Features

This DuckyScript leverages a staged PowerShell downloader to execute Invoke-Mimikatz in memory and send the harvested credentials back to a server. The success of this attack hinges on its quick execution and minimal visual footprint, making it a powerful demonstration of the USB Rubber Ducky's capabilities in penetration testing scenarios:

## Prerequisites

- **Operating System**: Tested on Windows 10 x64, version 22H2, Kali Linux 2023.4
- **Payload Studio**: Tested with Payload Studio Pro version 1.3.1
- **DuckyScript Version**: DuckScript 3.0 Advanced
- **Hardware**: USB Rubber Ducky (2022)

## Usage

### Step 1: **Install Apache and PHP (Kali Linux)**

Kali Linux comes with Apache and PHP available in its repositories. Check with the `-v` command.

1. **Update Your Package List:** `sudo apt update`
2. **Install Apache2**: `sudo apt install apache2`
3. **Install PHP**: Kali Linux usually includes PHP in its default installation. If it's missing for any reason, you can install it along with the common module for Apache by running: `sudo apt install php libapache2-mod-php`
4. **Start the Apache Service**: `sudo systemctl start apache2`
5. **Enable Apache to Start at Boot**: `sudo systemctl enable apache2`
    
    <aside>
    ⚠️ Remove Apache service from the system's startup sequence after completing exploit tests: `sudo systemctl disable apache2`
    
    </aside>
    

### **Step 2: Configure PHP Listener Script**

Prepare a PHP script to receive and save the credentials uploaded from the target PC. Host the PHP script as `listener.php` on your server:

1. **Prepare PHP Script**:
    - Navigate to the web root directory, typically `/var/www/html/` in Apache on Kali Linux.
    - Create a new directory for your project: `sudo mkdir /var/www/html/mimikatz_listener`
    - Create the `listener.php` file within this directory and insert your PHP code: `sudo nano /var/www/html/mimikatz_listener/listener.php`
    - Copy and paste your PHP listener script into the editor.
    - Save and exit the editor (`CTRL + X`, then `Y` to confirm, and `Enter` to save in nano).
2. **Accessing PHP Script Locally**
    - With Apache running, open a web browser.
    - Visit `http://localhost/mimikatz_listener/listener.php` to confirm that the Apache server is correctly serving your PHP file. This URL points to your PHP script. The actual interaction with this script will be from the payload executed by the USB Rubber Ducky.
    - If the PHP script doesn't execute as expected, check Apache's error logs for clues (`/var/log/apache2/error.log`).
        
        <aside>
        ⚠️ File Permissions:
        
        - The PHP script may not be creating files due to insufficient permissions on the directory where it's trying to write the files. Ensure that the Apache user (usually `www-data` on Debian-based systems like Kali Linux) has write permissions to the directory.
        - You can adjust the permissions with the following command, executed in the directory where you want the files to be created (`/var/www/html` for default Apache installations):
            
            `sudo chown -R www-data:www-data /var/www/html/*`
            `sudo chmod -R 755 /var/www/html/*`
            
        </aside>
        
3. **Reference Script in Payload:**
    - Adjust the `UPLOADURL` parameter in your payload: `DEFINE UPLOADURL http://localhost/mimikatz_listener/listener.php`
    - Replace `localhost` with your server's IP address (e.g. Kali Linux VM) if accessing from a different machine.

### Step 3: **Host the Invoke-Mimikatz Script**

Store and serve the Invoke-Mimikatz script from your Kali Linux web server to be executed by the target system. Invoke-Mimikatz script reflectively loads Mimikatz 2.2.0 in memory using PowerShell. Can be used to dump credentials without writing anything to disk.

1. **Download Invoke-Mimikatz Script:** 
    - Obtain the `Invoke-Mimikatz.ps1` script from the Nishang GitHub repository here: https://github.com/samratashok/nishang/blob/master/Gather/Invoke-Mimikatz.ps1
2. **Place Script in Web Server's Root Directory**:
    - Transfer the `Invoke-Mimikatz.ps1` script to the `/var/www/html` directory or a specific subdirectory within it, e.g., `scripts`.
        
        ```powershell
        sudo mkdir /var/www/html/scripts
        sudo cp path/to/Invoke-Mimikatz.ps1 /var/www/html/scripts/
        ```
        
3. **Verify Server Accessibility**:
    - Ensure that the Apache server is running. You can verify this by navigating to `http://localhost/scripts/Invoke-Mimikatz.ps1` from a web browser, which should display the contents of the Invoke-Mimikatz script.
4. **Reference Script in Payload:**
    - Modify your USB Rubber Ducky payload to include the URL to the Invoke-Mimikatz script hosted on your server. Adjust the `PS1URL` in your payload: `DEFINE PS1URL http://localhost/scripts/Invoke-Mimikatz.ps1`
    - Replace `localhost` with your server's IP address (e.g. Kali Linux VM) if accessing from a different machine.

## How It Works

### DuckyScript Payload:

- **Gain Administrative Access:** The script initiates by simulating keypresses to open a run dialog (`GUI r`), then types in a command to open an administrative PowerShell prompt. It uses the `Start-Process cmd -Verb runAs` command, automatically confirming any User Account Control (UAC) prompts to ensure administrative privileges.
- **Optional Obfuscation:** To reduce the visibility of the command prompt window during the attack, the script optionally resizes the window to a minimal footprint and changes the text and background colors to blend in with the background, making it harder for casual observers to notice.
- **Invoke-Mimikatz Execution:** With administrative access secured, the script downloads and executes Invoke-Mimikatz directly in memory. This is done through a PowerShell command that fetches the Invoke-Mimikatz script from a predefined URL (`PS1URL`) and executes it. The script is designed to dump credentials such as passwords from the memory of the compromised system without writing anything to the disk, minimizing its footprint and avoiding detection by disk-based antivirus software.
- **Exfiltrate Credentials:** After executing Invoke-Mimikatz and capturing credentials, the script uploads this sensitive data to a remote server specified by another URL (`UPLOADURL`). This step ensures that the attacker can access the harvested credentials from anywhere, provided they have access to the server.
- **Clean Up:** The script concludes by attempting to cover its tracks. It clears the run dialog history to remove evidence of the commands used during the attack. This is a crucial step in minimizing the attack's forensic footprint, although it's worth noting that sophisticated digital forensics could still potentially uncover the intrusion.
- **Server-Side Reception:** On the server side, a PHP script (`listener.php`) is prepared to receive and save the uploaded credentials. This script simply listens for incoming data and saves it in a file, appending the remote IP address and a timestamp for organization and future analysis.

### **Listener Script (PHP):**

- When the script receives a request, it first constructs a filename using the requester's IP address and the current date and time, appending `.creds` as the file extension. This ensures that each request is logged in a uniquely named file, preventing overwrites and making it easier to track when each set of data was received.
- It then captures the raw data sent in the request (such as credentials captured by the Invoke-Mimikatz script executed by the USB Rubber Ducky) using `file_get_contents("php://input")`.
- This data is saved to the newly created file with the constructed filename (`$file`).
- `$_SERVER['REMOTE_ADDR']`: This is a superglobal array in PHP that contains information about headers, paths, and script locations. The `REMOTE_ADDR` index holds the IP address of the visitor making the request to the script. This is used to identify where the request is coming from.
- `date("Y-m-d_H-i-s")`: The `date()` function formats a local date and time, and returns the formatted string. The format string `"Y-m-d_H-i-s"` generates a date and time in the format of Year-Month-Day_Hour-minute-second (for example, `2024-03-06_15-30-25`). This is used to timestamp the request, ensuring that each file created by the script is unique.
- `file_put_contents($file, file_get_contents("php://input"))`:
    - `file_put_contents()` writes a string to a file. If the file does not exist, it will be created. In this context, it's used to save the data received by the script.
    - `file_get_contents("php://input")` reads raw data from the request body. `php://input` is a read-only stream that allows you to read raw data from the request body. This is useful for reading the content of POST requests.
    - The script combines these functions to write the content received in the POST request to a file.

## Output Example

![DuckyScript Payload Results](/images/mimikatz_results.png)

- **Authentication Id, Session, User Name, Domain, Logon Server, Logon Time, and SID:** These fields provide context about the user session from which the credentials were extracted, including the username, the domain of the user, the logon server, and the time of logon.
- **NTLM, SHA1, DPAPI, CredentialKeys:** These fields represent different types of credentials. NTLM and SHA1 hashes are commonly used for network authentication. DPAPI is a Windows API for protecting and unprotecting data. `CredentialKeys` can include various forms of credential materials, including certificates or keys used by the system or applications.

## Contributing

If you have an idea for an improvement or if you're interested in collaborating, you are welcome to contribute. Please feel free to open an issue or submit a pull request.

## License

This project is licensed under the GNU General Public (GPL) License - see the [LICENSE](LICENSE) file for details.
