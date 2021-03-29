# Scolionet Design

## Introduction

Scolionet is a cloud base web application that receives patient data and 3D ultrasound images generated from individual Scolioscan mahines. Users of Scolionet are allowed to view and ananlyze the ulrasound images stored on Scolionet.

## Architectural Design

Scolionet is a cloud based system, which is composed of the following four major components:

- Web Server
- Web Service Server 
- Database Server
- Auxiliary Server

#### Web Server

The web application server serves the Web UI to the users. The Web UI will fetch patient information and examination data/images from Scolionet for view and analysis. 

#### Web Service Server

The Web Service Server is the center piece of Scolionet that provides core services for accessing the data in the database and services provided by other auxilary systems. Client applications connect to Scolionet and  retrieve information via Scolionet RESTful APIs.

New functionalites and enhancements can be added to the Web Service Server by developing new RESTful APIs. Moreover, the Web Service Server can be scaled up horizontally by adding more Web Service Servers   

#### Database Server

Scolionet employs replicated MongoDB for storing data and images. Data and images are sent to the primary MongoDB database, which the data and images are replicated to all secondary databases. This will increate data availability and will help to restore the database system whenever the primary database suffers a critical hardware failure. Currently, Scolionet has one primary database and one secondary database that are setup in separated locations. 

Snapshots of the data file of MongoDB will be made and archived. This will allow restoration of the whole database to the time point of the last snapshot, in case, a disastrous event happened.

#### Auxiliary Server

Auxilary servers are servers that provide auxilary functions/services to Scolionet. There will be two auxiliary servers to integrated to Scolionet:

- S3 compatible file storage - Scolionet will store version CD UVF in the database server. An option is provided for uploading the original version of UVF to this auxiliary server for archive purpose.
- Machine/Learning analytics engine - The analytics engine for processing and analyzing the data and images on Scolionet.

## Functional Components

Scolionet consists of different componets, ranging from data and imags generation to mangement and viewing appliocation. The following diagram depicts the interrelation of different components of Scolionet and a more detailed description of each components comes next. 

![Architecture](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210304142414903.png)

### Scolioscan/Scolioscan AIR

Scolioscan and Scolioscan AIR are the hardware devices to produce 3D ultrasound images of patient spines. It has an individual MS Access database, which is password protected, to store the patient data and examination records locally, and the 3D ultrasound images (UVF files) are stored in local file system.

Each Scolioscan machine will have a unique hardware ID for identification that is generated from a combination of different hardware IDs of the device. For example, Scolioscan AIR device uses the serial number of the ultrasound probe and the Intel Realsense camera to form its hardware ID. This hardware ID is stored in its local database and will be used for machine registration during the first connection to Scolionet. Details of the registration process will be discussed in the later section. 

### Scoliosync

The main function of Scoliosync is to upload data and images in each individual Scolioscan machine to Scolionet. It is a Windows service that will be started when Scolioscan boots up. The synchronisation process is a two stage process, namely, the track change process and upload process, which are scheduled to be executed periodically and separately. This allows synchronisation to be possible, even the Scolioscan machine might be in an offline/disconnected state.

### Scolionet Web UI

Users can use the Scolionet Web UI to access and review the patient data and images stored on Scolionet. Different users have different role and access control to access different functions/data/images of Scolionet.  

## Data and Images

### Data Structure

The following diagram shows the data structure of major data entities of Scolionet. It follows the data structure of Scolioscan database closely, which simplified the upload logic of Scoliosync. There are four types of data entities in Scolionet:

- Patient info and examination data
- User info data
- Machine info data
- Device info data

![Data Structure](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210329135930911.png)

### Patient Info and Examination Data

Patient Info data consists of the following components:

- Patient Info 

  It contains of the demographic data of the patients. Each patient info records represent one patient info record on a Scolioscan machine. Since Scolioscan machines are not connected, a patient may have one patient info record on each Scolioscan machine he/she has visited, and these patient records are not linked and are treated as different records when they are uploaded to Scolionet. 

  To resolve this limitation, Scolionet allows user to soft-link different patient info record of the same physical patient with **Patient ID**, so that patient exam records of the same physical patient can be relieved/reviewed in one single search.

- Patient Exam Record

  It represents a patient exam records on a Scolioscan machine and contains the information of the scanning examination taken by the patient. Moreover, it has the storage information of the images of the examination. 

  Record type is a new information added to patient exam record when it is uploaded to Scolionet. The purpose of record type is to differentiate examination records generated by other sources. By default, the record type for Scolioscan records is *Scolioscan*.

- Patient Image Info

  It is generated when the UVF and volume reconstruction layer bitmap images are uploaded to Scolionet. It contains the meta information of the images, such as, frame count, layer count, filename, so that status or alert (e.g., missing images) can be shown to users on the web UI.

- Patient Report

  It represents a patient report record on a Scolioscan machine and contains the measure report of an examination of a patient. User can generate multiple reports for an examination record, however, only the latest report will be display for review in the web UI. 

### User Role And Access Control

User role and access control are used for restricting access to different Scolionet Web UI functions and Scolionet data and images. Role information is added to user record when it is created. Access control information will be added to Scolionet records when they are uploaded or created. 

Currently, there are three user roles that can be assigned to users: 

- System Administrator

  Have access to all functions (Administration and Patient pages) and all data in Scolionet. Can create user of all roles.

- Clinic Admin

  Have access to all functions (Administration and Patient pages) and data with matched access control in Scolionet. Can create user with clinic administrator and user role.

- User

  Have access to Patient pages) and data with matched access control in Scolionet.

Access control is the key and lock for users to access patient data. The clinic ID is key and lock in the access control list in user and patient records. Users can have keys to different clinics. On the other hands, patient records usually have their own clinic ID as the lock in their access control list. Besides, the current design of the Web UI does not support access control management on the patient records.

The following diagram depicts the logic of access control. 

The user TMIL-ADMIN and BME-USER have the access key TMIL and BME respectively. Therefore, they can access patient records with corresponding access lock. 

The user BME-ADMIN has both BME and TMIL access key in his/her access control list, and therefore, he/she can access patient records with BME or TMIL access locks.  

![Role Access Example](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210304143851349.png)



## Scoliosync Functional Design

In following sections, we will go through various functions of the Scoliosync. 

#### Machine Registration

Before data and image can be uploaded to Scolionet, the Scolioscan machine will have to be first registered with Scolionet, before Scolionet accepts the data and images from the machine. The purpose of machine registration is to give an ID to the Scolioscan system, so that data and images will have a unique identification for storage and access. Besides, this will help Scolionet to differentiate legitimate uploads by recognising the registered machine information in the upload and reject all unsolicited uploads. The machine registration process is a two-step process that will take place automatically when the synchronisation function of Scoliosync in Scolioscan is enabled in the setting UI. The following depicts the two steps, which both of them could be completed in any order. 

![Machine Registration](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210308113537738.png)

- Device Registration

  When a device is sold to a customer, the hardware ID of the device will be registered in the device list of Scolionet, with the clinic ID and clinic name of the customer to be used in Scolionet. Once it is set, the clinic ID and clinic name could not be changed. 

- Machine Registration

  When the Scolioscan/Scoliosync software is installed, the Scolioscan system will generate and store the hardware ID in its database. The hardware ID is generated with reference to various serial number/ID used in the Scolioscan machine. For example, the hardware ID of a Scolioscan AIR machine is the combination of the serial number of the ultrasound probe and the Intel Realsense camera. 

  When the users enable the synchronisation function of Scoliosync in Scolioscan setting menu for the first time, Scoliosync will trigger a machine registration request to Scolionet, which contains the hardware ID.
  
  When Scolionet receives the machine registration request, it will check if the validity of the device's machine ID. If the request is legitimate, a new unique machine ID will be generated, and a success response with the clinic ID and machine ID will be returned to the machine, which the new IDs will be saved for future upload.

Machine registration binds to Scolioscan system, i.e., the software, but not the hardware device. Users can register one or more Scolioscan devices to be used with several Scolioscan system. The hardware ID of the Scolioscan device is used to obtain machine IDs for all Scolioscan systems. This arrangement allows users to use their Scolioscan devices efficiently. The following diagram demonstrates various scenarios of machine registration. All machines mentioned in the following scenarios had Scolioscan and Scoliosync installed, unless otherwise specified.

![image-20210308143550154](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210308143550154.png)

- TMIL produced four Scolioscan AIR devices and they were registered in the device list of Scolionet. Three of the devices were sold to two customers, and the clinic ID for the sold devices were updated as in the diagram. The purchase date in the device list could be used for the enablement date for using Scolionet.

- User PWH connected device AIR002 to a PC and enabled synchronisation to Scolionet. Scolionet received the registration request and returned clinic/machine ID PWH/PWH001 to the machine. Hardware ID was recorded in the machine list for record purpose, and would not affect data/image upload.
  - If by mistake, the device AIR002 was not registered in the device list or the purchase date was set to a later date (e.g. 2022-02-05), Scolionet will reject the registration request. 
  - Similarly, if a hacker forged a registration request with a non-existence hardware ID, the request would also be rejected.

- User PolyU BME purchased Scolioscan AIR device AIR001 and connected it to two machines and enabled synchronisation to Scolionet. Machine registrations were performed and machine ID POLYUBME001 and POLYUBME002 were generated and registered. Later, the user purchased another Scolioscan AIR device AIR003 for spare purpose. He/she connected the device to a machine to test the scan quality of the device. As no synchronisation was enabled on this machine, no registration was performed. 
  - In case the device AIR001 fails, user PolyU BME can connect the device AIR003 to the existing machines to continue operation. The device AIR001 would be sent back for RMA, which could be sent back to the original user, or sold to another customer (registration of device is needed). 

#### Data/Image Upload

After regsitration, Scoliosync will perform two scheduled tasks:

- Monitor Scolioscan Data Changes
- Upload Changes to Scolionet

These two tasks will be started automatically at the background once the Scolioscan machine is turned on and the machine registration has ben completed. The tasks will be repeated periodically, so that new examination records will be uploaded to Scolionet in a timely fashion.

![image-20210305113607140](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210305113607140.png)

##### Monitor Scolioscan Data Changes

1. Scoliosync will read the Scolioscan MS Access database for patient and examination records, namely, Patient Information, Patient Exam Record and Patient Report.
2. Then, it will check for discrepancy against the data in its staging database.
3. If there are differences between the data, the differences will be updated to the staging database and the corresponding records will be marked "need upload". 

##### Upload Changes to Scolionet

If an Internet connection is available, Scoliosync will process the records in the staging database.

1. The records that are marked "need upload" or "upload error" will be selected for upload.
2. An HTTP request will be created to send the updated data to Scolionet.
3. On Scolionet, the validity of the machine ID in the request will be checked. Request with invalid machine ID will be rejected.
4. If the machine ID is valid, the data in the request will be *upserted* (update or insert) to the database. 
5. Scolionet response the execution status to Scoliosync
6. If the request is completed successfully, Scoliosync will mark the records uploaded. Otherwise, it will mark the records with either "data error" or "upload error". 

For a Patient Examination Record, it will has UVF and volume cut bitmap files attached. Separate uploads for these will be arranged.

1. Resolve the path for the images.
2. If the image is not available, the corresponding records will be marked with "data error".
3. The images will be uploaded to Scolionet.
4. Scolionet will store or replace the received images in database and response to Scoliosync.
5. If the request is completed successfully, Scoliosync will mark the records with image uploaded. Otherwise, it will mark the records with "upload error". 

For UVF images, a conversion process might take place.

1. If the version of the UVF is not CD, it will be converted to a temporary copy in version CD.
2. The CD version will be uploaded.
3. If upload is successfully and "convert to CD" is configured, the original UVF will be zipped and the CD version will replace the original one

The two stage synchronisation approach as described in above allow changes to be accumulated in the staging database and synchronise to Scolionet when Internet is available. In addition, the *upsert* approach allows the data and images to be re-upload as long as it is needed. For example, if the staging database is deleted accidentally, a blank staging database could be created, so that all data and images would be synchronised again. The data and images that are in Scolionet will be replaced by identical values, while Scolionet users can still browse the data and images normally.  

## Scolionet Web UI Functional Design

In following sections, we will go through various functions of the Scolionet Web UI. 

### Login Dialog

User login is required to gain access to the functions as well as the data of Scolionet. The login dialog will be presented if the user is not logged on, or the login session has been expired. When a correct user ID and password is presented to Scolionet, a login session will be create on the backend of Scolionet and a session token will be stored in the web client. The session token will be presented in the request to Scolionet for obtaining data and images.

![image-20210223155649450](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210223155649450.png)

### Patient Search Page

Once logged on, the Patient Search page will be shown as following, where users can input search criteria to search for patients in interested. The Patient Search Page has three main components, namely, Header, Search Criteria Box and Search Result Box.

![Patient List Page](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210223180624540.png)

#### Header

The Header area contains the Main Menu switch that toggles the Main Menu for accessing other functions, as well as showing the logon information, namely the logon ID and the site logged on. 

#### Search Criteria

Users can specify the search criteria to search for patients to be reviewed and analysed. The following table describes the details of different search fields. 

| Search Field   | Description                                                  |
| -------------- | ------------------------------------------------------------ |
| SCN ID         | Specify the Scolioscan ID to be searched. Partial criteria can be used, e.g., POLYUBME will return all records which Scolioscan ID contains the text POLYUBME. |
| Patient ID     | Specify the patient ID to be searched. Partial criteria can be used. |
| First Name     | Specify the first name to be searched. Partial criteria can be used. |
| Last Name      | Specify the last name to be searched. Partial criteria can be used. |
| Telephone      | Specify the telephone to be searched. Partial criteria can be used. |
| DOB From/To    | Specify the range of the date of birth to be search. Click on the calendar icon to choose a date from the calendar selection. |
| Gender         | A radio button group for choose the gender to be searched.   |
| Clinic/Machine | Specify the clinic and machine of the records to be searched. The clinic and machine list will show the clinics and machines that is visible to the logged on users. |

#### Search Result Area

The search result box displays the search output of the criteria specified by the users. It has three main area, namely, search header, search result and selection buttons.

##### Search Header

The search headers do not only label the columns of the search result, it also provides filtering and sorting for manipulating the search output. Clicking on the up/down arrows next to the header labels sort the results in ascending or descending order. Type some text in the text fields below the header labels to filter the search output in a dynamic fashion. 

##### Search Result

It is the search result as specified by the search criteria. 

##### Selection Button

Users can click on the buttons to perform additional actions available. Currently, there are four actions associated to each record:

- Examination Record - Navigate to the Patient Details page for reviewing and analysing the examination records and reports of the patient. There is a number in the button showing the number of examination records for the patient.
- Upload Record - It will open a dialog for the users to upload X-Ray and Scolioscan data set. The number in the button shows the number of X-ray records associated to the patient.
- Scoliometer - Clicking the Scoliometer button will open a new page for making a Scoliometer scan. The number in the button shows the number of Scoliometer records associated to the patient. 
- Details - Click the button to open the record for more information about the patients.

### Patient Detail Page

The Patient Detail Page displays the examination records and examination report for the patient selected. There are video of b-mode images, cut layer images of volume reconstruction, measurement of angles, as well as X-ray images uploaded by users. 

![image-20210303154408651](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210303154408651.png)

##### Header and Patient Information

The header area is the same as the one shown the Patient List page. The Patient Information area shows the basic information of the selected patient for quick identification purpose.

##### Select an Examination Record

The Examination Record List displays the examination records of the patients which have the same Patient ID. The small file icons is the buttons for selecting the examination records to be displayed in the Selected Exam Record Images pane. The first examination record that has images attached will be selected automatically. A small file icon with an exclamation mark means the examination record does not have image attached to the record, which could due to the images has not been uploaded to Scolionet. In addition, the button is disabled and cannot be selected. 

![image-20210303161043361](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210303161043361.png)

Click on the small file icon to select the set of volume reconstruction layer images for the corresponding examination record. Click on the Examination Report Check Box to select the report images of the selected examination records. In the examination image pane, the volume reconstruction layer image and the report images will then be displayed accordingly as in the following. Four report images can be displayed simultaneously. If the fifth report is selected, the first selected report will be dismissed.

<img src="https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210303154534534.png" alt="image-20210303154534534" style="zoom:50%;" />

When an examination is selected, the user uploaded X-ray images associated with the selected examination record will be listed in the X-ray Record List. Users can select the X-Ray Record Check Box to display the set of X-Ray images, namely, namely, coronal, sagittal and miscellaneous image in the Examination Image Pane. If there were examination report images selected, they will be replaced with the x-ray images. Users can switch back to the examination report images by selecting another report image in the examination record list. 

<img src="https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210226141749016.png" alt="image-20210226141749016" style="zoom:50%;" />

For the X-ray images displayed, a measurement button for the X-ray image will be enabled. Clicking the button will open up a new window for measuring the angles on the X-ray image as shown in below. The angle measurement of X-ray is for convenience and the result will not be saved in the system. 

![image-20210303162002655](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210303162002655.png)

##### Examination Record Manipulation and Report Generation

When an examination record is selected, the volume reconstruction layer images and the b-mode video will be displayed as shown in below. There are various controls for manipulating the video and images. In addition, users can generate a report for the selected examination record.

![image-20210226151849395](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210226151849395.png)

The b-mode video is the sequence of b-mode images captured during an examination scan. Users can use the Frame Selector to browse through the b-mode images quickly. If the Probe Tracker is enabled, the Probe Indicator will be displayed in the volume reconstruction layer image. It shows the relative position of the ultrasound scanning probe with respect to the layer. In addition, the position of the probe is directly related to the current frame of the b-mode video.

Users can pause and resume the video with the play button, besides, the play speed can be adjusted with the speed control.

As discussed in previous section, an examination record has a set of volume reconstruction layer images and reports associated. Usually, a volume reconstruction layer image set has nine images, representing a cut layer at different depth of the reconstructed volume. Users can use the Layer Selector to select different layer for review and measurement.

An examination report is a record of angle measurement made on the selected examination record. If the examination record has a report associated, the measurement marks and angles will be displayed on top of the layer image that the measurement was made. Click on the Measure button toggle to the measurement mode to edit the measurement of the report or create a measurement report. The measurement marks will become movable and a delete mark will be shown for removing the measurement mark. Click on an empty area will add a new end point of a measurement mark. A maximum of five measurement marks or ten measurement points can be added to a measurement report. 

Users can Undo All Changes to revert to the original measurement. They can click on the Measure button to toggle off measurement and they will be asked whether to save or discard the measurement. If the users choose to save the measurement, a new examination report will be created. No change will be made on existing reports. 

<img src="https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210226170816024.png" alt="image-20210226170816024" style="zoom:50%;" />

After the report is generated, users can generate a PDF version of the report by clicking the Generate PDF button. The PDF will then be generated and displayed with the PDF Viewer of the browser, which the users can download or print the PDF report with the browser. The PDF reports are generated on demand and does not have to be saved in Scolionet.

<img src="https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210226175113204.png" alt="image-20210226175113204" style="zoom:50%;" />

### Administration Pages

The Administration Pages help the administrator of Scolionet to manage the system. Only users who are granted with "ADMIN" or "CLINIC ADMIN" access control can see the administration pages. Click on Main Menu > Administration to navigate to the Administration page. There are four administration areas supported by Scolionet, which are, 

-  Machine Management
- Patient Management
- User Management
- Upload Message Management
- Device Management

#### Machine Management 

It is an information screen that shows all the Scolioscan machines that have been registered with the Scolionet system. The Scolioscan machines will generate a health check heartbeat at a periodic period, which will be displayed in this page for monitoring purpose.

Users can search for some machines for management. When a search result us returned, the order of the search result can be manipulated by clicking the small triangles next to the column headers to toggle between ascending/descending/original orders. The header of the current sorted column will be highlighted. 

![machine management](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210329160155582.png)

During machine registration, Scolionet will assign a temporary name to the machine for display purpose. User can give a meaningful name in the Update Machine dialog, by clicking the Edit button. Besides, it is possible to stop upload from a specific machine by setting Upload Allowed to No in the dialog. This will cause Scolionet to reject upload from that machine temporarily. On Scolioscan side, a forbidden response for the upload request will be received and will be marked for retry in a later time.

![image-20210329160601964](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210329160601964.png)

#### Patient Management

The Patient Management page shared the same layout and portion of logic of the Patient List page. It allows users to search for a patient and then update the information of the patient. One important purpose is to allow users to group "patients" from different Scolioscan machines under one patient ID, so that examination record can be examined in one go.

![Patient Management](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210329161947672.png)

Click on the edit button on the right end of each patient will open up the patient edit page. When update is done, click on the Tick or Cross button to save or cancel the changes.

<img src="https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210302113248787.png" alt="image-20210302113248787" style="zoom:50%;" />

#### User Management

The User Management is used for creating and editing Scolionet Users. Use the search form to search for the users to be updated.

<img src="https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210329162154009.png" alt="image-20210329162154009" style="zoom:200%;" />

Click on the edit button or add button to bring up the Create/Update User form. Update or fill in the information of the users and then select the appropriate role and clinic/machine access accordingly. Finally, click of the tick button or cross button to save or cancel the changes made.

![image-20210329162253045](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210329162253045.png)

#### Upload Message Management

The Upload Message Management page shows the log message generated by Scoliosync running on each Scolioscan machine. The search form allows users to filter message logs from different Scolioscan machines.

![image-20210329162706260](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210329162706260.png)

#### Device Management

The Device Management is only accessible by Scolionet system administrators for assigning the devices to different customers. The administrators can search for machines to manage, or create a new device record for assignment. 

![image-20210329163138214](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210329163138214.png)

In the Create/Update Device dialog, administrators can assign the Clinic ID and Name for the new or existing devices. Scolionet will generate a list of Clinic IDs/Names that are currently used, so that administrators can create or choose Clinic ID/Name to be used easily. If the device has been sold, a purchase date could be specified. Moreover, the administrator can specify a date, when upload from the machine is allowed.

![image-20210329163331033](https://raw.githubusercontent.com/Joseph-Hui/scolionet-doc/main/uPic/image-20210329163331033.png)