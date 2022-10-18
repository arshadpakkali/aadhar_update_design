# AADHAR ADDRESS UPDATE DESIGN DOCUMENT

author: Arshad Pakkali (arshadpakkali@gmail.com)

## Current System [aadhar_service]

1. Data source (Eg. Postgres)
2. API Service (Eg. .NET API)
3. Client Applications (Eg. Web App, Mobile Apps)

## Requirements:

### Front End

- Client UI for users to Update Address

### Backend Service

Tech Stack :

- Kafka ( Aadhar, 138Cr population, needs to be HA )
- Nest.js (For writing Microservices, Simple MVC UI )
- Firebase (For having the Admins Login etc.)

1. Store the Data temporarily Until Admins Can Verify
2. Provide a UI Where Admins can view and Process User data
3. Incorporate APIs from Multiple Sources for Verification of documents
4. Incorporate API from the aadhar_service to update the User Data
5. Send Ack Back to User (Email, SMS, Push notification Etc.)

### Rough Design of the Architecture

[Architecture Diagram](https://drive.google.com/file/d/1jo0Wmwe4UgUhny2tTdqo2Kndn_DXaSOx/view?usp=sharing)

#### DTO'S

```
UpdateAddressDTO {
        ...UserIdentifier,
        ... addressStuff,
        uploaded_file_uri:string
    }


VerificationStatusDto{
    ...userIdentifier,
    ...adminIdentifier,
    status: boolean
    }


```

#### Front End

- Assuming we already have Clients for Users to View Aadhar

Flow:

1. user Logs in
2. FE Gets the keys to Connect to Kafka
3. The user Fills out the Update Address Form
4. Uploads the Address Proof
5. F.E uploads the Document to External Service and Builds the `UpdateAddressDto`
6. Once Validated F.E Produce a Message to Kafka in a topic ( queue_user_update-address )

#### Back End

We will have 3 Services

1. admin_verification_service

   1. login for Admin
   2. get a Message from kafka and the uploaded_file from s3
   3. Admin can verify the document by Querying with External services (DL, Ration etc)
   4. Once Verified Admin Creates a `VerificationStatusDto` and Publishes to Kafka on a topic (verification_status_update-address)

2. aadhar_update_service

   1. This Listens for Messages on verification_status_update-address
   2. If it's OK this interfaces with the Aadhar Db Maps the current data to whatever Aadhar DB wants and sends it

3. user_notification_service
   1. This also listens for Messages on verification_status_update-address and sends the Status
      via email/sms/Push with or without using external Services

Workspace:

Services - Nest.js

All 3 services can be a single Nest.js Microservice Module in a single Repo with an Entry Point File
Like `main.ts`
where we can Start any one/all of the Services by giving the Env/Args...
so we can write a single Dockerfile and have ENV variables Controlling the container Behaviour

1. Q: What could be the bottlenecks in such a system - for example, what should we do if too many requests are incoming which are greater than the capacity of the number of admins we have currently - For example if an admin can verify 1000 requests a day but they receive 5000 requests on a given day

   - A: as this Design has Scalability in mind the messages from clients go inside kafka and will be stored there untill a consumer takes it out of it. so The Admins can gradually process the Messages

2. Q: How could this system be abused?
   - A: By the user Publishing Redundant Messages to kafka
