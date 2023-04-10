# Development Configuration
Sets up a development copy of Dementia Care Works (DCW) environments


## Setup
Copy the `.env` file default:

    cp default.env .env

Modify the `.env` file as necessary. Lines that are not commented-out are required, commented lines are optional.


## Deploy
To pull the latest configured docker images, and re-deploy services as necessary, run the following command:

    docker-compose pull && docker-compose up --detach
