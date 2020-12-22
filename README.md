## Perform Secure FOTA Updates on a Nordic nrf9160-dk with AWS IoT Jobs

*About this Guide*
This guide is intended to provide an example of how to use AWS IoT Jobs to update a Nordic nrf9160-dk with ZephyrOS over HTTPS. By default, the examples provided by Nordic only give you the option to perform AWS FOTA over HTTP, so we built this walkthrough to provide you with a more secure option of completing your job downloads over S3 via HTTPS. 

## Overview
### Before You Begin / Considerations
This example is meant to be a guide and starting point for you launch your own FOTA updates with AWS IoT Jobs on the nrf9160-dk. You will want to adjust your security and job parameters to match your needs as the ones used here are 

This is a clone of the https://github.com/nrfconnect/sdk-nrf to complete an AWS IoT Job using HTTPS. For this example we are using version v1.2.0, please ensure you are using that codebase for the rest of this walkthrough.

### Cost
You will be responsible for the cost of all the AWS services used in this example which will be less than $5 USD as of current writing. Most nrf9160-dk kits come with an eSIM from iBASIS, if you do not have one,  please locate a suitable eSIM for your testing and have at least 500MB of data available for the purposes of this tutorial.

### Procedural Sections
To complete this guide, you will need to install the latest version of nRF Connect for Desktop (https://www.nordicsemi.com/Software-and-tools/Development-Tools/nRF-Connect-for-desktop).

Once you have the the nRF Connect for Desktop installed, you will need to follow theGetting Started (http://developer.nordicsemi.com/nRF_Connect_SDK/doc/1.4.0/nrf/getting_started.html) guide and go through the process specific to your guest operating system. You may proceed with the next steps to being modifying your custom application to complete AWS IoT Jobs over HTTPS once you have setup your working environment and cloned the v1.4.0 from https://github.com/nrfconnect/sdk-nrf. You will also need to install all required additional python dependencies (http://developer.nordicsemi.com/nRF_Connect_SDK/doc/1.4.0/nrf/gs_installing.html#installing-additional-python-dependencies).

### Walkthrough
After you have the repository cloned and have completed the Getting Started guide provided by Nordic, you will register your nrf9160-dk with AWS IoT services. Follow the current AWS IoT Guide (http://developer.nordicsemi.com/nRF_Connect_SDK/doc/1.4.0/nrf/include/net/aws_iot.html) provided by Nordic so that you can quickly get your nrf9160-dk modem updated and connect to AWS IoT services. Once you have completed the walkthrough, you should now have your device security connected with the certificates you created during the creation process. 

First, you will need to make a copy of the application aws_fota located in nrf/samples/nrf9160/ and name the copy aws_fota_secure. Now you will proceed with following guide provided by Nordic AWS FOTA (http://developer.nordicsemi.com/nRF_Connect_SDK/doc/1.4.0/nrf/samples/nrf9160/aws_fota/README.html#nrf9160-aws-fota), however all the changes that you make will be in the application aws_fota_secure. Before proceeding to the next step, you should flash your application to the device and confirm you see the following output when your device starts up by using the terminal in nRF Connect - LTE Link Monitor, where ClientID is the name of your IoT device in AWS IoT.

    IPv4 Address XX.XX.XX.XX
    client_id: CLIENTID [mqtt_evt_handler:182] 
    MQTT client connected!

### Test the functionality of AWS IoT Jobs over HTTP before changing to HTTPS
Update the APP_VERSION to a new version in your KConfig file for your aws_fota_secure application. Once done, perform a build using either the Segger utility or west from command line. Once you have built your application, you will retrieve the “aws_fota_secure/build/zephyr/app_update.bin“ file from the new build and you will upload it to a public S3 bucket along with a complete AWS IoT Job document, for ease of testing, I am providing a sample job document below, you will need to update the BUCKETNAME next to ”host“ to your public bucket name. 


    {
    "operation": "app_fw_update",
    "fwversion": "v1.0.1.http",
    "size": 181124,
    "location": {
    "protocol": "https:",
    "host": "BUCKETNAME.s3.amazonaws.com",
    "path": "app_update.bin"
    }
    }


### Create your first AWS IoT Job
Login to your AWS console and navigate to AWS IoT, then click on Jobs on the left hand menu under “Manage”. You will now have a blue button on the right hand side of your screen named “Create” which you will click, then select Create a Customer Job. Pick a unique job id type such as the fwversion you will be pushing.

Important: Before creating your job, open the nRF Connect - LTE Link Monitor and be ready to watch the terminal output. 

You will then be able to select your target device under “Select devices to update” and select your job document under “Add a job file”. All other settings may remain default, click Next. For Step 2 of the job creation, leave everything set to default and click Create. Now switch over to your Terminal in the LTE Link Monitor and watch the ouput, you should see something like the following and once the device restarts, you will see the newly updated application version. 


    [mqtt_evt_handler:235] PUBACK packet id: 65277I: 
    Connecting to aaarcr-zephyr.s3.amazonaws.com
    I: Downloading: app_update.bin [0]AWS_FOTA_EVT_START, job id = jobv1-0-1-http
    I: Downloaded 2048/187904 bytes (1%)
    I: 2 Sectors of 4096 bytesI: alloc wra: 1, 900I: data wra: 1, 38c
    I: Erasing page at offset 0x00087000I: Downloaded 4096/187904 bytes (2%)
    I: Downloaded 6144/187904 bytes (3%)
    I: <truncated for brevity> 
    SPM: NS MSP at 0x20027f08SPM: NS reset vector at 0x217e1SPM: prepare to jump to Non-Secure image.
    *** Booting Zephyr OS build v2.4.0-ncs1  
    ***MQTT AWS Jobs FOTA Sample, version: v1.0.1.http
    Initializing bsdlib Initialized bsdlib
    LTE Link Connecting ...LTE Link Connected!
    IPv4 Address XX.XX.XX.XX
    client_id: CLIENTID[mqtt_evt_handler:182] 
    MQTT client connected!

Your AWS IoT Job should now show a status of complete. We will now proceed with changes that will allow the updates to be downloaded from Amazon S3 over HTTPS instead of HTTP, which it does in the current example. 

To confirm this was done over HTTP instead of HTTPS, you can enable file logging for your S3 bucket and then review your logs, this is an example of what an HTTP download would look like on the app_update.bin file, what you will notice is three dashes at the end, the last one indicates if TLS was used, click here for more details on Amazon S3 Log Formatting (https://docs.aws.amazon.com/AmazonS3/latest/dev/LogFormat.html).

    REST.GET.OBJECT app_update.bin "GET /app_update.bin HTTP/1.1" 206 - 2048 187904 11 11 "-" "-" - p8yY7bV/7ZblOvFDgTomeRkCOvfEm97mcI2cHYKYr//yUY2YT+teixY48hmydBSqN6uQJ47jqjA= - - - 


### Changing to an HTTPS S3 Download
First, we are going to make a few changes to the file nrf/subsys/net/lib/download_client/src/download_client.c 

At line 75, you will find a section for the “socket_sectag_set” where you will need to make the following changes, please make sure that you update the BUCKETNAME below (line 88) and change verify = REQUIRED to OPTIONAL. The full section of code is listed here so you can compare or replace:

    static int socket_sectag_set(int fd, int sec_tag)
    {
    int err;
    int verify;
    sec_tag_t sec_tag_list[] = { sec_tag };

    enum {
        NONE = 0,
        OPTIONAL = 1,
        REQUIRED = 2,
    };

    verify = OPTIONAL;
    char hostn[] = "BUCKETNAME.s3.amazonaws.com"; //Add your own bucket name here.
    err = setsockopt(fd, SOL_TLS, TLS_PEER_VERIFY, &verify, sizeof(verify));
    if (err) {
        LOG_ERR("Failed to setup peer verification, errno %d", errno);
        return -errno;
    }

    LOG_INF("Setting up TLS credentials, tag %d", sec_tag);
    err = setsockopt(fd, SOL_TLS, TLS_SEC_TAG_LIST, sec_tag_list,
             sizeof(sec_tag_t) * ARRAY_SIZE(sec_tag_list));
    if (err) {
        LOG_ERR("Failed to setup socket security tag, errno %d", errno);
        return -errno;
    }

    err = setsockopt(fd, SOL_TLS, TLS_HOSTNAME, hostn, sizeof(hostn) - 1);
    if (err) {
        LOG_ERR("Failed to setup socket HostName, errno %d", errno);
        return -1;
    }
    LOG_INF("All Goood in Socket SEC TAG SET + Host Name Set");

    return 0;
    }


Next, we need to make another small change to the nrf/subsys/net/lib/aws_fota/src/aws_fota.c

For this change, we are going to update the “int sec_tag” to the security tag ID you used when you installed the TLS certificates on the device to connect to AWS.


    int sec_tag = 1234567;


Now save both files, and rebuild and flash to your device. Once that is done, make a change to your KConfig file in your nrf/samples/nrf9160/aws_fota_secure folder, build and upload the app_update.bin and a new job file to match the version, following the same process above as you did for the HTTP version. 

Once you have submitted your new IoT job, you can review your S3 logs or Cloudfront logs and you will see that the download occurred over HTTPS, similar to this sample.


    2020-12-10    18:35:31    PHX50-C2    1552    XXX.XXX.XXX.XXX    GET    BUCKETNAME.s3.amazonaws.com    /app_update.bin    206    -    -    -    -    Hit    _BEJQcsfJHlclVT2rwwiXVlGXNdo1Q9PU7I-VafCmJfIwX4d1ugq9g==    d3zzcki9218zp.cloudfront.net    https    127    0.002    -    TLSv1.2    ECDHE-RSA-AES128-SHA256    Hit    HTTP/1.1    -    -    39075    0.002    Hit    application/macbinary    1024    175104    176127 
    2020-12-10    18:35:31    PHX50-C2    1231    XXX.XXX.XXX.XXX    GET    BUCKETNAME.s3.amazonaws.com   /app_update.bin    206    -    -    -    -    Hit    l0uDfj_O6elh8vGlPIIDImeKyOOAT1sdPgW9KzOJP3KLTxg98LSPkA==    d3zzcki9218zp.cloudfront.net    https    127    0.002    -    TLSv1.2    ECDHE-RSA-AES128-SHA256    Hit    HTTP/1.1    -    -    39075    0.001    Hit    application/macbinary    704    176128    176831


### Source Code
The source code for this project is available from nrfconnect via GitHub: https://github.com/nrfconnect/sdk-nrf and was modified in the example above for the purposes of this walkthrough. 

Official Nordic documentation at:

* Latest: http://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest
* All versions: http://developer.nordicsemi.com/nRF_Connect_SDK/doc/


### Conclusion
Now that you have completed this walkthrough, you should have the ability to push AWS IoT jobs from your AWS IoT Console and perform secure updates over HTTPS to your nrf9160-dk. You can use this knowledge as a baseline to further integrate your nrf9160 based IoT devices into other AWS IoT services and manage them securely in the field. 

### Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

This guide is intended for demo purposes only and considerations should be made for the security policies to be more restrictive instead of what is created here. Always start with a least privileged model and only allow trusted access based upon your own security policies.  

### License

This library is licensed under the MIT-0 License. See the LICENSE file.
