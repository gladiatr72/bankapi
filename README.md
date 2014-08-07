# bankapi

BankAPI is a OpenPGP compliant secure distributed messaging system.

The system is built on top of PostgreSQL and uses its pgcrypto contrib module.

Messages are encrypted and signed using RSA public key cryptography.

Each bank generates a RSA key-pair consisting of a public and a secret key.

The public keys and API URLs are exchanged between the banks.

The task of exchanging public keys and API URLs is preferrably done over the SWIFT network,
by sending them in a normal MT999 SWIFT FIN-message.
This allows banks to trust the validity of the public key and API URL,
as the origin of SWIFT messages can be trusted.

## Highlights
- RSA-encryption
- SHA512
- JSON-RPC / HTTPS
- Linux / PostgreSQL / Apache
- Open source / MIT-license
- Real-time request/response
- Instant delivery receipt
- Local archiving of files in PostgreSQL

## Protocol
To send a message, the sending bank encrypts/signs a message using Create_Message(),
and Get_Message() to get the actual chipertext content of the message.
The chipertext is the delivered to the receiving bank by calling it's Receive_Message()
API method, accessible at the API URL provided by the receiving bank.
The API must adhere to the JSON-RPC standard and accept HTTPS POST.

The Receive_Messge() function returns a DeliveryReceipt, which is a cryptographic proof
the sender can verify against the receiving bank's public key, to be certain the message
was delivered successfully.

The sending bank calls the Decode_Delivery_Receipt() with the DeliveryReceipt as input,
to verify its validity.

## Database tables and columns

### Banks
- BankID : Unique identifier for the bank, preferrably the SWIFT BIC (PRIMARY KEY)
- Protocol : API procotol, example "https"
- Host : API host, Internet domain name, example "bank.com"
- Port : API port, Internet network port, example 443
- Path : API path, example "/api"
- Datestamp : Date/time when added to the table

### Files
- FileID : SHA512 hash of the Plaintext (PRIMARY KEY)
- Plaintext : The content of the file (UTF8)
- Datestamp : Date/time when added to the table

### Keys
- MainKeyID : Signature key (PRIMARY KEY)
- SubKeyID : Encryption key
- PublicKeyring : Contain public keys
- SecretKeyring : Contains secret keys (only set for your own bank, NULL for others)
- BankID : The BankID which the keys belong to
- Datestamp : Date/time when added to the table
- PrimaryKey : Only set to TRUE for one row per BankID, other keys are still valid, but this key is used when creating new messages

### Messages
- MessageID : SHA512 hash of the Cipherdata (PRIMARY KEY)
- FileID : The hash of the file send in the message
- FromBankID : The bank who sent the message
- ToBankID : The bank who received the message
- Cipherdata : The encrypted and signed message
- DeliveryReceipt : The encrypted and signed delivery receipt
- Datestamp : The date/time when the message was created
- Delivered : The date/time when the message was delivered

## Database functions

### Create_Message(Plaintext, FromBankID, ToBankID)
Writes the Plaintext to Files, if it doesn't already exists.
Looks up the secret signature main key for FromBankID, i.e. your bank.
Looks up the public encryption sub key for ToBankID, i.e. the other bank.
Encrypts and signs the Plaintext into a Message, by calling Encrypt_Sign().
Writes the Message to Messages.
Returns the MessageID, which is a SHA512 hash of the Message.

### Get_Message(MessageID)
Returns the ASCII armored PGP message for the MessageID.
This is the data to be HTTPS POSTed to the ToBank's Receive_Message() API method.

### Receive_Message(Ciphertext text)
Decrypts and verifies the signature of the Message.
Writes the Plaintext to Files.
The FromBankID and ToBankID is derived from the keys used to encrypt/sign the message.
Writes the Message to Messages.
If the message was successfully verified, a DeliveryReceipt is generated by
encrypting and signing the FileID (the hash of the Plaintext).
The DeliveryReceipt is returned to the sender who called Receive_Message(), i.e. to the sending bank.

### Decode_Delivery_Receipt(DeliveryReceipt text)
Decrypts and verifies the DeliveryReceipt, where the keys involved and the Plaintext
should match a row in Messages. If so, the Message has been successfully delivered,
and the Messages.DeliveryReceipt and Messages.Delivered columns are set.


## Installing and building .deb packages for installation
The repository includes scripts for building debian/ubuntu packages for the
bankapi and the required pgcrypto extensions (in PostgreSQL) in order to run
the code. 

If you are running a different version of PostgreSQL then 9.3 then replace the
version in the commands below with the appropriate version. If you are running
the default version in Debian you can omit the version extension of the
packages names.

- sudo apt-get build-dep postgresql-9.3
- sudo apt-get install postgresql-server-dev-9.3
- make

This should produce two deb files, one for bankapi and one for the pgcrypto extension.

### bankapi-0.1.deb
Package contains the command line tools for sending messages
(/bin/sendbankmessage), SQL needed for creating the bankapi in postgres and
installes a CGI as an endpoint for receving incoming communication messages.

The installation of the module installs and activates the CGI script. It will
be available under http://localhost/bankapi .

The installation does not finish the installation of the database but leaves
this as a manual task. Follow the following steps for a default database
installation:
- cd /usr/share/bankapi
- sudo -u postgres createdb bankapi
- sudo -u postgres psql --dbname=bankapi --single-transaction --no-psqlrc --file=install.sql
- Make sure www-data user can connect to the bankapi database, this can for
  instance be done by adding the following line where appropriate in the active
  pg_hba.conf file: local www-data bankapi peer
- To optionally install the test bank data: sudo -u postgres psql --dbname=bankapi --single-transaction --no-psqlrc --file=testdata/index.sql

### postgresql-pgcrypto-signatures-9.3.deb
This is an extension to the pgcrypto PostgreSQL extension. The extension only
installs with the extended functions, all functions normally in pgcrypto is
still accessed from the standard pgcrypto extension. The extension is called
pgcrypto_signatures.

