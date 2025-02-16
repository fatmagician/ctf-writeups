# Silver Platter
This is a writeup for the TryHackMe room [Silver Platter](https://tryhackme.com/room/silverplatter). It involved finding user and root flags on a vulnerable web server.
The only tools I used for this challenge were Nmap, Burp Suite, the TryHackMe Attack Box, Google, and basic CLI commands. 

## Recon/Scanning
1.	We’ll start by visiting the site and conducting some basic recon. The home page looks like this:
   
![image](https://github.com/user-attachments/assets/21cf887c-bac7-4aec-b062-e48de0be3cb7)

After exploring a few of the links on the home page, the one that jumps out the most as potentially helpful is the “Contact” page. It lists a username “scr1ptkiddy” that their project manager uses on something called Silverpeas. I googled Silverpeas and found out that it is an open-source collaboration web app. 

![image](https://github.com/user-attachments/assets/770f6609-f276-4525-b9e3-5ad5eaf7e204)

2.	**Nmap Scan**. We’ll dig more into Silverpeas later, but for now we’ll conduct an Nmap scan to see what ports are open. Nmap shows ports 22, 80, and 8080 are open.

![image](https://github.com/user-attachments/assets/b5256243-f3a0-476e-b966-d66132d145a7)

3.	**Research Silverpeas.** From the [Silverpeas website](https://www.silverpeas.org/), we can get a few useful pieces of information:
    * The default admin credentials are username: SilverAdmin, password: SilverAdmin
    * It can be accessed through the url `http://localhost:8000/silverpeas`. 

![image](https://github.com/user-attachments/assets/1bff98c6-a132-40df-ba60-9ef7ed310799)

When we use this url scheme with our target IP and port 8080 (<target IP>:8080/silverpeas), it brings us to a Silverpeas login portal.

![image](https://github.com/user-attachments/assets/e8fb2470-6ef6-4071-ab3e-56b28b696646)

4.	**Try default admin credentials.** I tried the default SilverAdmin/SilverAdmin credentials to login, but these didn’t work.
   
5.	**Research Silverpeas vulnerabilities.** In a previous step, we found the username “scr1ptkiddy” for the manager’s Silverpeas username, but we don’t have a password. Before trying to brute force the password, which might take a long time, I searched Google for Silverpeas exploits. I found two different sites listing Silverpeas vulnerabilities that might be useful:
    * [CVE-2024-36042](https://gist.github.com/ChrisPritchard/4b6d5c70d9329ef116266a6c238dcb2d) – This is an authentication bypass which lets you log in as any user without a password as long as you know a valid username (which we do).
    * [Multiple CVEs from Rhino Security](https://rhinosecuritylabs.com/research/silverpeas-file-read-cves/) – 8 different CVEs that might come in handy.

## Gain a Foothold
6.	**Use the authentication bypass vulnerability to login as sr1ptkiddy.** Using Burp Proxy, we’ll follow the methodology explained in the authentication bypass vulnerability to login as scr1ptkiddy. Start BurpSuite and intercept a request to login as scr1ptkiddy. It doesn't matter what password you use, because we're about to remove that field from the request.

![image](https://github.com/user-attachments/assets/aaeef509-a13b-4d8d-ac75-d9b69cfd4933)

Before forwarding the request, remove the “Password” field so that the Login field reads “Login=scr1ptkiddy&DomainID=0”. Forward this request, and we’re successfully logged in as scr1ptkiddy!

![image](https://github.com/user-attachments/assets/9ae84ca3-cb0e-4a82-8ee8-af038103cddf)
 
7.	**More recon.** Now that we’re logged in as scr1ptkiddy, explore the site a bit. One thing we find under “Directory” is a list of users:

![image](https://github.com/user-attachments/assets/36438d7f-7a8c-484a-9116-2c2030484844)

It looks like the Administrator account might still be using the SilverAdmin username, as indicated by the “silveradmin@localhost” under the Administrator profile. If you want to you can use the same technique that we used to login as scr1ptkiddy to login as SilverAdmin and have administrator privileges in Silverpeas. Although this is not necessary. The next vulnerability we'll exploit can be done as any user. 

![image](https://github.com/user-attachments/assets/acadf211-ce14-47a5-931f-be2406b1abd3)

8.	**Research additional vulnerabilities.** Now that we’re logged in to Silverpeas, let’s explore some additional vulnerabilities from the Rhino Security site. One that proves useful is CVE-2023-47323, which is a broken access control vulnerability that allows you to read all messages by navigating to the url: `http://localhost:8080/silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=[messageID]`.
   
9.	**Exploit broken access control.** Manually enumerate through different messageIDs to see if we can find anything. We find one for messageID 6 at `http://<target IP>:8080/silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=6`. This reveals a username and SSH password for the user “tim”. 

![image](https://github.com/user-attachments/assets/4e80923a-17cc-43d9-ae2e-5887f67c7166)

10.	**Connect via SSH.** Now let’s connect as “tim” via SSH using the password found in that message.

![image](https://github.com/user-attachments/assets/8ffc701c-ac93-45b1-947a-e8b15d156fa2)

11.	The user flag is located in /home/tim:

![image](https://github.com/user-attachments/assets/46ab08ff-8690-4432-acf4-8ee14d983913)

## Escalate Privileges
12.	Next, it’s time to escalate our privileges. Use the `id` command to display info about tim’s group names:

![image](https://github.com/user-attachments/assets/227309f7-2aa8-4adf-b4c1-989fb8993b4f)

13.	A Google search shows that the “adm” group is used for monitoring functions, and gives tim permission to read log files, such as those stored in `/var/log`. We’ll search through the `/var/log` directory to see if we can find any passwords using the command `grep -Ri ‘password’ /var/log`. (Note, in the screenshot below I initially only used the -R option, which did not find what I needed. I ran the command again, adding the -i option, which makes the search term non-case sensitive, and this found what I needed. However, I forgot to take a screenshot of it.)
 
  * -R option allows us to search through a directory
  * -i option makes the search term non-case sensitive
  * ‘password’ is our search term
  * /var/log is the target directory we’re searching in

![image](https://github.com/user-attachments/assets/1d610c59-5e7a-4ea6-ac1f-45018aa7a20d)

14.	There’s a lot of results to sort through, but after looking for a minute or two it looks like we found a password for the user “tyler”: 

![image](https://github.com/user-attachments/assets/25cce905-c376-49fc-874b-e1fba15a2917)

15.	Using tyler’s password, we’ll now switch users to tyler:

![image](https://github.com/user-attachments/assets/d794224e-0169-4594-bd90-37c808ce1f14)

16.	Use `sudo -l` to see what commands tyler can run as sudo, and confirm he can run ALL commands:

![image](https://github.com/user-attachments/assets/d3dd7d72-9b31-44f3-9024-d93af4447d18)

17.	Change directories to the root directory.
    
18.	Use `sudo su` to change to root user, which will let us read files in the /root directory.

19.	Search for the root.txt file using the command `find . -name root.txt 2>/dev/null`

![image](https://github.com/user-attachments/assets/b421d860-bfd6-4424-a03f-0a1a1773d603)

20.	Read the root.txt file to get the root flag and complete the challenge.

![image](https://github.com/user-attachments/assets/c375fc34-2307-49fc-954f-edca05be4c0e)

## Conclusion
We found both flags! In looking through the vulnerabilities Rhino Security found for Silverpeas, there’s probably a few different ways to gain access and elevate privileges. In particular, [CVE-2023-47322](https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2023-47322) is a CSRF vulnerability that leads to privilege escalation and could likely be used as an alternate way to escalate privileges.

Thanks for reading, and good luck with this challenge!
