# Junit

## File > Settings > Plugins

Ensure Code Coverage for Java being installed

<p>
  <img width="267" height="198" alt="image" src="https://github.com/user-attachments/assets/8c71d983-c524-4110-b03c-22780ca8b6ea" />
  <img width="515" height="195" alt="image" src="https://github.com/user-attachments/assets/ea1cc47c-d640-4cfe-ba63-3cd4c1a52fc9" />
</p>

At pom.xml, set org.jacoco, then set prepare-agent and report to generate report once maven clean verify.

Maven verify = report + check stages

<p>
  <img width="335" height="339" alt="image" src="https://github.com/user-attachments/assets/ba5940a3-f87c-48cf-bd8f-a23766bca49d" />
  <img width="253" height="340" alt="image" src="https://github.com/user-attachments/assets/0d7188f7-1677-4ea4-8efb-fa2ac7421eca" />
</p>

After maven clean + verify, go to project files\target\site\jacoco\index.html, then it will display the code coverage for Junit.

<p>
  <img width="350" height="119" alt="image" src="https://github.com/user-attachments/assets/5c156edd-1198-4a80-b669-d20eaddadcb9" />
  <img width="581" height="241" alt="image" src="https://github.com/user-attachments/assets/222905bc-1857-477c-9c89-3021c258f437" />
</p>

Click 1 of the package, then u can start design your Junit based these class to ensure it pass.

<p>
  <img width="439" height="87" alt="image" src="https://github.com/user-attachments/assets/14d2d55d-9f1f-4cb9-967d-311cfdf80d47" />
  <img width="490" height="204" alt="image" src="https://github.com/user-attachments/assets/9f353a2f-b83d-488e-9cb7-5d79b1238194" />
</p>

---

## How to set Junit path

Example DynamicQrCodeUtil.java, u see the package path

<p>
  <img width="401" height="48" alt="image" src="https://github.com/user-attachments/assets/384d48cb-1ff4-4b05-be8c-e59ec5bf4472" />
</p>

Then your Junit path will be (File name behind have to put behind)

If folder not exist have to create by your own

`src/test/java/com/nets/nps/qr/common/utils/DynamicQrCodeUtilTest.java`

<p>
  <img width="319" height="278" alt="image" src="https://github.com/user-attachments/assets/e2054b79-0db6-4c5a-b682-a9d82c19df63" />
</p>

---

## How to write JUnit

Basically, write a function based on function u developed for backend, then insert value and use Junit API like assertEquals/assertTrue to pass the code coverage.

<p>
  <img width="690" height="270" alt="image" src="https://github.com/user-attachments/assets/9b84fa9a-62c6-4b9f-9fa4-72139eab5fce" />
</p>

- assertEquals()
- assertNotNull()
- assertTrue()

Throw to AI (chatgpt/doubao), let it generate test cases to improve time, while u need to understand how it works

It increased now.

<p>
  <img width="940" height="58" alt="image" src="https://github.com/user-attachments/assets/70d93567-0a3e-4be5-a1e1-7288f4f48d61" />
</p>

---

## Code coverage not able to reach 100%

Its normal, some part is impossible to reach 100%. But u have to ensure it went above 80%. This is not Junit issues, its code-structure, unless u adjust the code

<p>
  <img width="940" height="156" alt="image" src="https://github.com/user-attachments/assets/f4510ebf-7613-4f46-9930-808df1437f04" />
</p>

### Public class cant pass

<p>
  <img width="367" height="170" alt="image" src="https://github.com/user-attachments/assets/83726cf3-0ff8-435f-a796-8d85fa3ed85b" />
</p>

Create a constructor at Junit. Then use assertDoesNotThrow with class name.

<p>
  <img width="700" height="59" alt="image" src="https://github.com/user-attachments/assets/dfd8860f-819e-403d-a8da-cd07280900b0" />
</p>

### Super()

<p>
  <img width="545" height="114" alt="image" src="https://github.com/user-attachments/assets/014513a5-6d5c-4ad4-aca9-10e1a583ef15" />
</p>

Declare the object, then use assertNull and getters

<p>
  <img width="361" height="197" alt="image" src="https://github.com/user-attachments/assets/fd836e65-6749-4d5e-b19c-1b58997f4d63" />
</p>

### When function has specific declaration (string/int)

<p>
  <img width="372" height="121" alt="image" src="https://github.com/user-attachments/assets/220c031c-5c9b-4e50-9516-2a85f115bfc8" />
</p>

Map value inside, then use assertNotNull/assertEquals/assertNull

<p>
  <img width="493" height="185" alt="image" src="https://github.com/user-attachments/assets/e0c16b63-2f50-4574-9cc0-14b4069c9bab" />
</p>

### This.variable

<p>
  <img width="501" height="58" alt="image" src="https://github.com/user-attachments/assets/d5e272ac-94da-411a-b7c3-f4d8077c3bc8" />
</p>

Declare a value for the variable, then use assertEquals to pass

<p>
  <img width="500" height="175" alt="image" src="https://github.com/user-attachments/assets/11a9063f-8b2c-4494-be9a-d5a4c11e4374" />
</p>

### toString()

<p>
  <img width="510" height="226" alt="image" src="https://github.com/user-attachments/assets/0d7c553a-cdc4-419e-a6a4-03ca191cc976" />
</p>

Based on if-else statement insert suitable value, then use String to store toString() value, then use assertEquals to compare

<p>
  <img width="517" height="269" alt="image" src="https://github.com/user-attachments/assets/72c00086-085e-4d61-8a1a-fe9ebc252949" />
</p>

---

## If-else statement fulfill but still not pass

### Scenario 1 (null/isEmpty())

<p>
  <img width="727" height="88" alt="image" src="https://github.com/user-attachments/assets/7fe98008-9b19-44f8-ac81-cb9986ba892b" />
</p>

Because null = 1 scenario, “” = 1 scenario. Have to separate at your test case.

If null, use null, if isEmpty, use “”

<p>
  <img width="429" height="200" alt="image" src="https://github.com/user-attachments/assets/607b2e48-8a33-4b32-b2b6-9c1894b401cc" />
  <img width="432" height="201" alt="image" src="https://github.com/user-attachments/assets/09d93b03-0a11-4eea-aaf9-2a950f8a8715" />
</p>

### Scenario 2 (Impossible condition)

<p>
  <img width="940" height="93" alt="image" src="https://github.com/user-attachments/assets/8635123f-f96a-4df7-84b7-e63c8217b5e9" />
</p>

`message.getCommunicationData() == null AND message.getCommunicationData() != null`

You wont complete this condition, because u put value it will hit !null, without value it will null. Unless u adjust the code (but too risky since its wrote by sg side)

### Scenario 3 (Private class + hidden logic)

<p>
  <img width="593" height="253" alt="image" src="https://github.com/user-attachments/assets/92d4aca7-b9a1-406b-92c0-64f6e9119188" />
</p>

calculateSignature() is private, so it cannot be called directly and is correctly tested through the public sign() method. The SHA 256 algorithm is guaranteed to exist on any compliant JVM, so NoSuchAlgorithmException will never occur and md can never be null at runtime. Because of this, the checks for md == null and hashcode == null are defensive but unreachable, meaning no test can cover those branches. The remaining JaCoCo warnings are therefore phantom branches caused by bytecode analysis, not missing tests or faulty logic.

### Scenario 4 (Null + isEmpty condition)

When both condition (null + isEmpty) is combined, just accept that its impossible to solve. Because Junit not able to identify null + isEmpty, so when 1 of equvation like if null and! isEmpty, u literally can’t get pass over it.

Somehow it passed, but u have to use “” to complete

<p>
  <img width="599" height="64" alt="image" src="https://github.com/user-attachments/assets/26a0976a-6168-4d58-a4e7-196b04397f8f" />
  <img width="599" height="188" alt="image" src="https://github.com/user-attachments/assets/ee694df3-a038-439b-ab61-f8ab7637cf62" />
</p>

### Scenario 5 (Thread.currentThread().join())

It is impossible to test because once JVM runs, then u able to use thread, but u cant force it to shut down so the Junit is impossible

### Scenario 6 (Dummy value)

<p>
  <img width="820" height="95" alt="image" src="https://github.com/user-attachments/assets/7569d406-ce0d-4430-9a79-430f899485ab" />
</p>

When u want to insert dummy value, use Mockito provided features

<p>
  <img width="481" height="183" alt="image" src="https://github.com/user-attachments/assets/ee6dc0eb-9f65-4d21-8cdf-369ec268955c" />
</p>

---

## If Junit limit can’t go above 80%, have to provide valid reason

### Npx2-notification

<p>
  <img width="940" height="183" alt="image" src="https://github.com/user-attachments/assets/8c10fead-047e-4e7e-96c0-1874069d6d9a" />
</p>

Com.nets.upos.notification.expection cant go above 80%, CustomHTTPRetryPolicy.java  because pass this Junit have to pop MessageHandlingException or the method crash, but Junit don’t have such feature

<p>
  <img width="718" height="222" alt="image" src="https://github.com/user-attachments/assets/cc2a2663-47bd-4420-a067-86501293a5f8" />
</p>

com.nets.upos.notification.communication cant go above 80%, HTTPWebClientMessenger because NoSuchAlgorithmException, the logic will auto implement a value in it, hardly to be null.

MalformedURLException, uncertain what scenario will pop this exception, WebClientRequestException cant catch because it called at reactor thread instead of main thread

<p>
  <img width="700" height="272" alt="image" src="https://github.com/user-attachments/assets/98e823ad-f983-4e1a-9e2b-39fdadf81c6b" />
</p>

com.nets.upos.notification cant go above 80%. NotificationApplication.java because it using thread. In order to pass thread.currentthread().join() is to shutdown JVM, but once shutdown JVM your junit will stop running

<p>
  <img width="776" height="127" alt="image" src="https://github.com/user-attachments/assets/f12f8ee7-6a72-4f71-a710-8e0efd5c932d" />
</p>

com.nets.upos.notification.service, HeaderEnricherService.java because it’s a private class, and MessageDigest will automatically assign value to default, so NoSuchAlgorithmException is impossible to hit.

<p>
  <img width="739" height="101" alt="image" src="https://github.com/user-attachments/assets/be52803f-fc49-488c-b079-3c58f78f396b" />
</p>

For ValidationService.java, Junit is not able to identify null + isEmpty when both are same condition, impossible to hit because there is 1 condtion data.getCommunitcationData == null and data.getCommunicationData != empty, it will not be pass

<p>
  <img width="940" height="89" alt="image" src="https://github.com/user-attachments/assets/15715186-f9a9-4b15-8c88-2d894e558bee" />
</p>

com.nets.upos.notification.dto, SMSDTO.java having same issue which is null + isEmpty scenario.

<p>
  <img width="940" height="113" alt="image" src="https://github.com/user-attachments/assets/20d2db2d-b9a4-4ec9-a708-b02ba96f72fc" />
</p>

---

## JaCoCo didn’t generate result (index.html)

Because u didn’t bring import Junit5 at dependencies

<p>
  <img width="450" height="117" alt="image" src="https://github.com/user-attachments/assets/f8a22c90-6de8-43d1-8236-22a92bd9539a" />
</p>

### Junit4 is not supported in JDK21 + Springboot 3.5.11

<p>
  <img width="513" height="85" alt="image" src="https://github.com/user-attachments/assets/fc0a5778-a8ca-4c51-8f77-aa5c05016ff8" />
</p>

Org.springframework.boot (spring-boot-starter-test) will automatically install Junit 5, so u can remove Junit 4 dependencies now, they are not compatible with JDK21 + Springboot 3.5.11

https://docs.spring.io/spring-boot/reference/testing/index.html

<p>
  <img width="571" height="197" alt="image" src="https://github.com/user-attachments/assets/cc5dc0b1-67f0-4e9a-8626-ad392acda2ed" />
</p>

### Dependencies that can be removed

<p>
  <img width="348" height="142" alt="image" src="https://github.com/user-attachments/assets/44295793-3630-4d2a-a9ca-586260dc88d3" />
  <img width="334" height="196" alt="image" src="https://github.com/user-attachments/assets/5156259f-92a1-45e5-8133-94f8334e74c7" />
</p>

### Having Junit issues after remove

<p>
  <img width="288" height="74" alt="image" src="https://github.com/user-attachments/assets/4a8be25c-3040-4998-9bb2-74cbdc5e3f37" />
</p>

Actually only have to swap import name, then u can continue to use

<p>
  <img width="940" height="74" alt="image" src="https://github.com/user-attachments/assets/9af8b37a-311e-4ce6-bc72-c6fa344e4ec0" />
</p>

---

## Junit4 vs Junit5

- Org.junit.Assert.assertEquals/assertNotNull swap to org.junit.jupiter.api.Assertions
- Org.junit.Before/Test swap to org.junit.jupiter.api.BeforeEach/Test
- @Before swap to @BeforeEach

<p>
  <img width="910" height="237" alt="image" src="https://github.com/user-attachments/assets/a005a6a5-7242-4620-950b-020af08041d0" />
  <img width="907" height="69" alt="image" src="https://github.com/user-attachments/assets/c29eef1f-d2d9-4c81-8fea-a9af91947e1a" />
</p>
