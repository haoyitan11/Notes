# Java Environment setup for Intelij
## Only use 1 version of JDK
Download your specific JDK version and setup your system environment
https://www.oracle.com/java/technologies/javase/jdk11-archive-downloads.html
<img width="940" height="162" alt="image" src="https://github.com/user-attachments/assets/ca141ee4-fd19-4891-90aa-57e76a95e6df" /> <br>

After install done, find ENV on search bar
<img width="375" height="269" alt="image" src="https://github.com/user-attachments/assets/69b1c7ec-3fb1-42c4-8916-7e7f749489ed" /> <br>
 

Create new, set Java_Home
<img width="254" height="279" alt="image" src="https://github.com/user-attachments/assets/b7d4be1b-3150-4fc2-ab89-bd7ed0b6d5bd" /> <br>
<img width="542" height="162" alt="image" src="https://github.com/user-attachments/assets/8c526209-abc0-4d94-9bac-0f9ecc5cd852" /> <br>


then go to path, set %JAVA_HOME%\bin inside
<img width="600" height="271" alt="image" src="https://github.com/user-attachments/assets/80567ed6-2f71-4c48-b43f-8c54e1630fff" /> <br>
<img width="294" height="324" alt="image" src="https://github.com/user-attachments/assets/4a63e3b5-04ee-49a8-b5b2-b4743f66367e" /> <br>

Remember to click apply

Go Terminal , write command java -version to check java jdk 
<img width="595" height="218" alt="image" src="https://github.com/user-attachments/assets/dfd39714-c04a-4bad-b110-f8fcaab64099" /> <br>

## Swap multiple JDK version with script
1.Create 2 scripts for 2 JDK , 1 is JDK8 , another one is JDK11
Jdk8.bat
echo off
set JAVA_HOME=C:\Program Files\Java\jdk1.8.0_202 
set PATH=%JAVA_HOME%\bin;%PATH%
echo Java Version 
java -version

jdk11.bat
echo off
set JAVA_HOME=C:\Program Files\Java\jdk11.0.15
set PATH=%JAVA_HOME%\bin;%PATH%
echo Java Version 
java -version
<img width="548" height="241" alt="image" src="https://github.com/user-attachments/assets/82ab9e24-377b-429a-a7f2-6d7ab456d5d6" /> <br>

2.Set file location on system variable PATH , so u can swap two JDK at command
<img width="561" height="239" alt="image" src="https://github.com/user-attachments/assets/ba83649f-0a9e-40d0-8deb-7b7c72820f00" /> <br>
<img width="301" height="239" alt="image" src="https://github.com/user-attachments/assets/a326af48-bfe6-4d45-bd2b-10f6044beb53" /> <br>
 
3.Type it on command prompt
 <img width="465" height="275" alt="image" src="https://github.com/user-attachments/assets/b59cc728-b7d2-4650-b766-5b29d8f7c8a7" /> <br>

