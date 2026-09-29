# Junit
File > Settings > Plugins 
Ensure Code Coverage for Java being installed.
<p>
<img width="267" height="198" alt="image" src="https://github.com/user-attachments/assets/a09d5491-af22-4515-ae37-cf95cd9c4e59" />
<img width="515" height="195" alt="image" src="https://github.com/user-attachments/assets/1d045214-f0a5-4caf-ba1b-4beaf356b800" />
</p>

At pom.xml, set org.jacoco, then set prepare-agent and report to generate report once maven clean verify.
Maven verify = report + check stages
<p>
<img width="335" height="339" alt="image" src="https://github.com/user-attachments/assets/4e7fde1e-e028-46e2-a763-95338c382229" />
<img width="253" height="340" alt="image" src="https://github.com/user-attachments/assets/5ba1d851-9889-4e09-9656-bed31a7354d1" />
</p>

After maven clean + verify, go to project files\target\site\jacoco\index.html, then it will display the code coverage for Junit.
<p>
<img width="350" height="119" alt="image" src="https://github.com/user-attachments/assets/b64805f2-9d7d-49cd-bc1b-56f001daf5d8" />
<img width="581" height="241" alt="image" src="https://github.com/user-attachments/assets/deedb2e7-f95a-4c07-af7a-d9d20d24d253" />
</p>

Click 1 of the package, then u can start design your Junit based these class to ensure it pass.
<p>
<img width="439" height="87" alt="image" src="https://github.com/user-attachments/assets/1053c67a-4fbd-4605-868c-c4d1dcb104ec" />
<img width="490" height="204" alt="image" src="https://github.com/user-attachments/assets/b5134504-25db-407c-883c-f9c26ad7410b" />
</p>   

How to set Junit path
Example DynamicQrCodeUtil.java, u see the package path
<p>
<img width="401" height="48" alt="image" src="https://github.com/user-attachments/assets/8b271fdc-27ea-45bc-969c-2f618836f6a5" />
</p>

Then your Junit path will be (File name behind have to put behind)
If folder not exist have to create by your own
src/test/java/com/nets/nps/qr/common/utils/DynamicQrCodeUtilTest.java
<p>
<img width="319" height="278" alt="image" src="https://github.com/user-attachments/assets/a3c57a3a-4656-468a-afc6-f9447f86d885" />
</p>

Your test project path structure must be same as project path structure, else JaCoCo cannot scan the test coverage
