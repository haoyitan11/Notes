# Java Environment Setup for IntelliJ

## 1. Use a Single JDK Version

Download the specific JDK version you want to use and set up the system environment.

[Download JDK 11](https://www.oracle.com/java/technologies/javase/jdk11-archive-downloads.html)

<p>
  <img width="940" height="162" alt="JDK Download" src="https://github.com/user-attachments/assets/ca141ee4-fd19-4891-90aa-57e76a95e6df" />
</p>

After the installation is complete, search for **Environment Variables** from the Windows search bar.

<p>
  <img width="375" height="269" alt="Environment Variables" src="https://github.com/user-attachments/assets/69b1c7ec-3fb1-42c4-8916-7e7f749489ed" />
</p>

### Configure `JAVA_HOME`

Create a new environment variable and set the variable name to `JAVA_HOME`.

<p>
  <img width="254" height="279" alt="JAVA_HOME" src="https://github.com/user-attachments/assets/b7d4be1b-3150-4fc2-ab89-bd7ed0b6d5bd" />
</p>

<p>
  <img width="542" height="162" alt="JAVA_HOME Configuration" src="https://github.com/user-attachments/assets/8c526209-abc0-4d94-9bac-0f9ecc5cd852" />
</p>

### Configure `Path`

Go to the system `Path` variable and add:

```text
%JAVA_HOME%\bin
```

<p>
  <img width="600" height="271" alt="Path Configuration" src="https://github.com/user-attachments/assets/80567ed6-2f71-4c48-b43f-8c54e1630fff" />
</p>

<p>
  <img width="294" height="324" alt="Path Configuration" src="https://github.com/user-attachments/assets/4a63e3b5-04ee-49a8-b5b2-b4743f66367e" />
</p>

Remember to click **Apply** to save the changes.

### Verify the JDK Installation

Open a new terminal and run the following command to check the installed JDK version:

```bash
java -version
```

<p>
  <img width="595" height="218" alt="Java Version" src="https://github.com/user-attachments/assets/dfd39714-c04a-4bad-b110-f8fcaab64099" />
</p>

## 2. Switch Between Multiple JDK Versions

You can create `.bat` scripts to switch between different JDK versions.

In this example, we will create two scripts:

* JDK 8
* JDK 11

### JDK 8

Create a file named `jdk8.bat`:

```bat
@echo off
set JAVA_HOME=C:\Program Files\Java\jdk1.8.0_202
set PATH=%JAVA_HOME%\bin;%PATH%
echo Java Version
java -version
```

### JDK 11

Create a file named `jdk11.bat`:

```bat
@echo off
set JAVA_HOME=C:\Program Files\Java\jdk11.0.15
set PATH=%JAVA_HOME%\bin;%PATH%
echo Java Version
java -version
```

<p>
  <img width="548" height="241" alt="JDK Switching Scripts" src="https://github.com/user-attachments/assets/82ab9e24-377b-429a-a7f2-6d7ab456d5d6" />
</p>

### Add the Script Location to `Path`

Add the folder containing the JDK switching scripts to the system `Path` variable.

<p>
  <img width="561" height="239" alt="System Path" src="https://github.com/user-attachments/assets/ba83649f-0a9e-40d0-8deb-7b7c72820f00" />
</p>

<p>
  <img width="301" height="239" alt="System Path Configuration" src="https://github.com/user-attachments/assets/a326af48-bfe6-4d45-bd2b-10f6044beb53" />
</p>

### Switch JDK Versions

Open Command Prompt and run the corresponding script.

For JDK 8:

```cmd
jdk8
```

For JDK 11:

```cmd
jdk11
```

<p>
  <img width="465" height="275" alt="JDK Version Switching" src="https://github.com/user-attachments/assets/b59cc728-b7d2-4650-b766-5b29d8f7c8a7" />
</p>
