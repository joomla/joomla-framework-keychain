## The Keychain Package [![Build Status](https://travis-ci.org/joomla-framework/keychain.png?branch=master)](https://travis-ci.org/joomla-framework/keychain)

[![Latest Stable Version](https://poser.pugx.org/joomla/keychain/v/stable)](https://packagist.org/packages/joomla/keychain)
[![Total Downloads](https://poser.pugx.org/joomla/keychain/downloads)](https://packagist.org/packages/joomla/keychain)
[![Latest Unstable Version](https://poser.pugx.org/joomla/keychain/v/unstable)](https://packagist.org/packages/joomla/keychain)
[![License](https://poser.pugx.org/joomla/keychain/license)](https://packagist.org/packages/joomla/keychain)

The keychain provides a way to securely store sensitive information such as access credentials or any other data.

The system relies on four files:

* a public key
* a private key
* a passphrase file
* a keychain file

The *passphrase file* is generated by using the *private key* to encrypt the passphrase. This is so that the passphrase file can be decrypted by the *public key* without requiring the knowledge of the passphrase for the private key. This means it can be deployed onto a server without requiring manual intervention or a passphrased stored plain text on disk. Because of this the public key should not be stored in a repository and should be stored on servers in a protected location.

The *keychain file* is the actual valuable contents. It is encrypted using the passphrase stored in the passphrase file (which itself is decrypted using the public key).

This provides a balance between not storing credentials plain text but also making the system reasonably independent.

A good example of where the keychain is useful is where some code needs to establish a connection with another server or service using some access credentials (usually a username and password, but any number of authentication credentials could be used); using clear text credentials in the code, which is probably stored on a relatively public code repository, can be avoided by storing the credentials in an encrypted data file that the keychain can read.

### Key Storage

You can store the private key in a code repository but you **MUST NOT** commit the **public key**. Doing so will compromise the security of the keychain (you will need to regenerate the private key if you accidentally do commit it).

### Classes

#### `Keychain\Keychain`

##### Construction

The `Keychain` class extends `Registry`. There are no changes to the constructor's argument list so optional initialisation with data can be done in the normal way.

```php
use Joomla\Keychain\Keychain;

// Create a keychain.
$keychain1 = new Keychain;

$keychain2 = new Keychain(array('username' => 'foo', 'password' => 'bar'));
```

##### Usage

A `Keychain` object operates in the same way as a `Registry` object. What `Keychain` provides is a way to load data from, and store data to an encrypted data source.

When using this class, the private and public keys must already exist on the server. The third required element is the passphrase file and the following example shows how to create it.

```php
use Joomla\Keychain\Keychain;

// Create a keychain object.
$keychain = new Keychain;

// The passphrase/password should not be stored in any code repository.
$passPhrase = 'the Pass Phrase';
$privateKeyPwd = 'the Private Key Password';

// The paths to keychain files could come from the application configuration.
$passPhrasePath = '/etc/project/config/keychain.passphrase';
$privateKeyPath = '/etc/project/config/keychain.key';

$keychain->createPassphraseFile($passPhrase, $passPhrasePath, $privateKeyPath, $privateKeyPwd);
```

The passphrase file will generally be created using the Keychain Management Utility (see next section) on the command line so that neither the passphrase, nor the private key password are stored in clear text in any application code.

Likewise, initial data is probably already created in a keychain data file (again, using the Keychain Management Utility and the `create` command). The following example shows how to load the keychain data:

```php
use Joomla\Keychain\Keychain;

// Create a keychain object.
$keychain = new Keychain;

$keychainFile = '/etc/project/config/keychain.dat';
$passPhrasePath = '/etc/project/config/keychain.passphrase';
$publicKeyPath = '/etc/project/config/keychain.pem';

$keychain->loadKeychain($keychainFile, $passPhrasePath, $publicKeyPath);

$secureUsername = $keychain->get('secure.username');
$securePassword = $keychain->get('secure.password');

$conn = connect_to_server($secureUsername, $securePassword);
```

The `Keychain` object can manipulate data as if it was a normal `Registry` object. However, an additional deleteValue method is provided to strip out registry data if required.

Finally, the saveKeychain method can be used to save data back to the keychain file.

### Keychain Management Utility

The keychain management utility `/bin/keychain` allows you to manage keychain resources and data from the command line.

#### Usage

```sh
keychain [--keychain=/path/to/keychain]
	[--passphrase=/path/to/passphrase.dat] [--public-key=/path/to/public.pem]
	[command] [<args>]
```

##### Options

Option                                 | Description
-------------------------------------- | -----------------------------
`--keychain=/path/to/keychain`         | Path to a keychain file to manipulate.
`--passphrase=/path/to/passphrase.dat` | Path to a passphrase file containing the encryption/decryption key.
`--public-key=/path/to/public.pem`     | Path to a public key file to decrypt the passphrase file.

##### Commands

Command                         | Description
------------------------------- | -------------------------------
`list [--print-values]`         | Lists all entries in the keychain. Optionally pass --print-values to print the values as well.
`create entry_name entry_value` | Creates a new entry in the keychain called "entry\_name" with the plaintext value "entry\_value". NOTE: This is an alias for change.
`change entry_name entry_value` | Updates the keychain entry called "entry\_name" with the value "entry\_value".
`delete entry_name`             | Removes an entry called "entry\_name" from the keychain.
`read entry\_name`              | Outputs the plaintext value of "entry\_name" from the keychain.
`init`                          | Creates a new passphrase file and prompts for a new passphrase.

#### Generating Keys

On a command line with openssl installed (any Mac OS X or Linux box is suitable):

```sh
openssl genrsa -des3 -out private.key 1024
```

This command will generate a new private key in the file "private.key". This can be then used to create a new public key file:

```sh
openssl rsa -in private.key -pubout -out publickey.pem
```

This will use the private key we just created in private.key to output a new public key into the file publickey.pem.

#### Generating Keys with Certificates

If you need to generate keys with certificates (exact details will vary from system to system), on a command line with openssl installed:

```sh
openssl req -x509 -days 3650 -newkey rsa:1024 -keyout private.key -out publickey.pem
```

This will create a new private key in the file private.key and a new public key in the file publickey.pem. You will be asked for a passphrase to secure the private key. and then prompted for information to be incorporated into the certificate request:

```
Country Name (2 letter code) [AU]: US
State or Province Name (full name) [Some-State]: New York
Locality Name (eg, city) []: New York
Organization Name (eg, company) [Internet Widgits Pty Ltd]: Open Source Matters, Inc.
Organizational Unit Name (eg, section) []: Joomla! Framework
Common Name (eg, YOUR name) []: Joomla Credentials
Email Address []: platform@joomla.org
```

Once this is done there will be a private.key and publickey.pem file that you can use for managing the passphrase file.

#### Initialise a new passphrase file

This step requires that you have already generated a private key (and assumes the `bin/keychain` file is executable and in your lookup path). The following command will initialise a new passphrase file:

```sh
keychain init --passphrase=/path/to/passphrase.file --private-key=/path/to/private.key
```

This will prompt for two things:

* the passphrase to store in passphrase.file;
* and the passphrase for the private key.

It will create a new file at `/path/to/passphrase.file` replacing any file that might be there already.

#### Create a new entry in the keychain

This step requires that you have already generated the private key and the passphrase file. The following command will create or update an entry in the keychain:

```sh
keychain create --passphrase=/path/to/passphrase.file --public-key=/path/to/publickey.pem --keychain=/path/to/keychain.dat  name  value
```

An existing keychain file will attempt to be loaded and then key name will be set to value.

#### Create a new public key from private key

If you know the passphrase for the private key but have lost the public key you can regenerate the public key:

```sh
openssl rsa -in private.key -pubout -out publickey.pem
```

This will use the private key in the file `private.key` and output a new public key to `publickey.pem`. If the private key has a passphrase on it, you will be prompted to enter the passphrase.


## Installation via Composer

Add `"joomla/keychain": "2.0.*@dev"` to the require block in your composer.json and then run `composer install`.

```json
{
	"require": {
		"joomla/keychain": "2.0.*@dev"
	}
}
```

Alternatively, you can simply run the following from the command line:

```sh
composer require joomla/keychain "2.0.*@dev"
```
