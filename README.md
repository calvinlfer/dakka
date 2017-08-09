# Java and NodeJS Image

A compact docker image based on Alpine Linux with:

- JRE 8 (8u111)
- NodeJS 7 (7.7.1)
- kms-env 
- bootstrapping behavior for retrieving host ip /port within container

In addition to having node and java installed, this image
comes with [kms-env](https://github.com/ukayani/kms-env) installed.
If you pass in environment variables encrypted using `kms-env`. 
The image will automatically decrypt them. 

In order for decryption
to work, your docker container must be running on an ec2-instance with
a role that has access to AWS KMS (and the master key used for encryption).
