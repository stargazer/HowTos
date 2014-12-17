# SSL Certificates

In order to issue an SSL Certificate for a domain (say ``*.domain.com``) using a Certificate Authority(``CA``) 
like COMODO, we need to:

* Generate a Certificate Signing Request (``CSR``) 
       			 
    	openssl req -new -newkey rsa:2048 -nodes -keyout domain.com.key -out domain.com.csr
		
  This generates a ``key`` file and a ``csr`` file

* We send the csr file to the CA, and we keep the key file for ourselves
* The CA will give us a 4 files in return
 
  * ``STAR_domain.com.crt``: The actual SSL Certificate
  * ``COMODORSADomainValidationSecureServerCA.crt``, ``COMODORSAAddTrustCA.crt``, ``AddTrustExternalCARoot.crt``: Used to form the Certificate chain
  which I explain later.

Now, we can use these files to configure our servers. 

## Configure Apache
Apache uses the following directives to configure the SSL certificates:

* ``SSLCertificateFile``: Needs to point to the actual SSL Certificate that we received from the CA, which is 
  ``STAR_domain.com.crt`` in our case
* ``SSLCertificateKeyFile``: Needs to point to the key file we got when we generated the CSR, which is ``domain.com.key`` in our case.
* ``SSLCACertificateFile``: Most clients have a large database of CA certificates and can actually verify which CA has signed
  the server's SSL certificate. Sometimes though the client doesn't have the certificate for an intermediate CA that has signed
  the SSL certificate. That's why we provide the server with a Chain file, which contains the certificates of the CAs that have 
  signed our SSL certificate and those that have signed their certificates. In our case we should concatenate files ``COMODORSADomainValidationSecureServerCA.crt``,
  ``COMODORSAAddTrustCA.crt`` and  ``AddTrustExternalCARoot.crt``, store the output to a separate file, and make sure that the 
  Apache directive ``SSLCACertificateFile`` points to that file.

  	
    	cat COMODORSADomainValidationSecureServerCA.crt COMODORSAAddTrustCA.crt AddTrustExternalCARoot.crt > domain.com.cer

  So ``SSLCACertificate`` should point to file ``domain.com.cer``


Here's the Apache configuration that would enable the use of the SSL certificates we obtained:

	<VirtualHost *:443>
		...
		...
		SSLCertificateFile		/path/to/STAR_domain.com.crt
		SSLCertifucateKeyFile	/path/to/domain.com.key
		SSLCACertificateFile	/path/to/domain.com.cer
		...
	</VirtualHost>


## Configure the Google App Engine

In order to configure the Google App Engine to use SSL on a custom domain, we need to upload the SSL Certificate file and the Key file.
Since we can only upload one certificate file, this file has to contain both the actual SSL Certificate, as well as the chaining mechanism. 
We therefore have to concatenate the file ``STAR_domain.com.crt`` as well as the 3 we had concatenated before:

	cat STAR_domain.com.crt COMODORSADomainValidationSecureServerCA.crt COMODORSAAddTrustCA.crt AddTrustExternalCARoot.crt > domain.com.cer

So we upload these 2 files ``domain.com.cer`` and ``domain.com.key``, to the Google App Engine.
     

## Sources

* http://billpatrianakos.me/blog/2014/04/04/installing-comodo-positive-ssl-certs-on-apache-and-openssl/
* http://superuser.com/questions/347588/how-do-ssl-chains-work
* http://serverfault.com/questions/382633/difference-between-sslcertificatefile-and-sslcertificatechainfile
* http://httpd.apache.org/docs/2.2/mod/mod_ssl.html#sslcacertificatefile


