# Find Usages (Check where value interact within functions)

One VO setup done, another VO need to insert these value also. You can use find usage, to see how the data retrieved.

Since you know this function will retrieve with JSON format, then u can find usage the function, then select the deepest one, see how it interact with another VO.

<p>
  <img width="883" height="239" alt="image" src="https://github.com/user-attachments/assets/70a0bd0d-4aa3-46ea-a5ed-9d4817f31400" />
</p>

<p>
  <img width="940" height="175" alt="image" src="https://github.com/user-attachments/assets/4c5ebcc3-cf4b-4fd7-8c61-cc1242c6f3bd" />
</p>

Found out the deepest function , pymtTrxnVO retrieve value based on hostSourceList, then right click middleSystemPosting , u found out pymtTrxnVO is from hostSourceList.get(0)

<p>
  <img width="940" height="202" alt="image" src="https://github.com/user-attachments/assets/4a85435b-06f8-43b8-b0ef-081fe4c78155" />
</p>

Then u have to see hostSourceList

<p>
  <img width="940" height="187" alt="image" src="https://github.com/user-attachments/assets/51a8e45b-ae24-48c9-aa76-edac361c4830" />
</p>

hostSourceList come from getSourceList, then right click middleSystemPosting function again, then find usage

<p>
  <img width="540" height="153" alt="image" src="https://github.com/user-attachments/assets/07b36982-1c0c-41ef-8202-7c890d28e48b" />
</p>

Then u have to find where dataContainer.getSourceList come from, its from paymentHostMapVO, have to get in

<p>
  <img width="865" height="334" alt="image" src="https://github.com/user-attachments/assets/1d1dbe35-3541-4a2f-b029-a65dc4ba515e" />
</p>

Have to see where paymentHostMapVO come from. It is from paymentHostList

<p>
  <img width="715" height="139" alt="image" src="https://github.com/user-attachments/assets/5c1ff3be-a797-481d-ae30-52fc6e59506f" />
</p>

Have to see where paymentHostList come from. It is from prepareSendTransactionToMiddleSystemHost.

<p>
  <img width="940" height="144" alt="image" src="https://github.com/user-attachments/assets/b20874ce-8038-44f8-baad-ebb343a67cd5" />
</p>

Found out the function is using p_trxn_pymt_middle_sys_dtl_s.

<p>
  <img width="896" height="256" alt="image" src="https://github.com/user-attachments/assets/65d29c3e-c213-449e-a1c5-fac5a26fe34e" />
</p>

<p>
  <img width="900" height="183" alt="image" src="https://github.com/user-attachments/assets/3423b75f-01f6-4b46-8fdc-457e56200732" />
</p>
