https://www.youtube.com/watch?v=jsD_lAE8Odg
https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md

cfssl gencert --help 
cfssl ls
cfssl print-defaults csr
cfssl print-defaults config

cfssl gencert -initca ca-csr.json
This gives you all 3 certificates but we want each certificate in a separate file.
cfssljson --help
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

2023/03/31 15:40:50 [INFO] generating a new CA key and certificate from CSR
2023/03/31 15:40:50 [INFO] generate received request
2023/03/31 15:40:50 [INFO] received CSR
2023/03/31 15:40:50 [INFO] generating key: rsa-2048
2023/03/31 15:40:51 [INFO] encoded CSR
2023/03/31 15:40:51 [INFO] signed certificate with serial number 595408172640289122691190423472144634592996419026
# we now have 3 files generated
ls
ca.csr  ca-key.pem  ca.pem

https://lapo.it/asn1js/
look at the certificate generated
xclip -selection clipboard -i < 'ca.pem'

openssl x509 -in ca.pem -text

cfssl gencert --help
cfssl gencert -ca cert -ca-key key [-config config] [-profile profile] [-hostname hostname] CSRJSON

cfssl gencert -ca cert -ca-key key


https://blog.cloudflare.com/introducing-cfssl/

How CFSSL Makes Certificate Bundling Easier
If you are running a website (or perhaps some other TLS-based service) and need to install a certificate, CFSSL can create the certificate bundle for you. Start with the following command:

$ cfssl bundle -cert mycert.crt

This will output a JSON blob containing the chain of certificates along with relevant information extracted from that chain. Alternatively, you can run the CFSSL service that responds to requests with a JSON API:

$ cfssl serve
This command opens up an HTTP service on localhost that accepts requests. To bundle using this API, send a POST request to this service, http://localhost:8888/api/v1/cfssl/bundle, using a JSON request such as:

{
    "certificate": <PEM-encoded certificate>
}

CloudFlare’s SSL service will return a JSON response of the form:

{
    "result": {<certificate bundle JSON>},
    "success": true,
    "errors": [],
    "messages": [],
}
(Developers take note: this response format is a preview of our upcoming CloudFlare API rewrite; with this API, we can use CFSSL as a service for certificate bundling and more—stay tuned.)

If you upload your certificate to CloudFlare, this is what is used to create a certificate bundle for your website.

To create a certificate bundle with CFSSL, you need to know which certificates are trusted by the browsers you hope to display content to. In a controlled corporate environment, this is usually easy since every browser is set up with the same configuration; however, it becomes more difficult when creating a bundle for the web.

Different Certs for Different Folks
Each browser has unique capabilities and configurations, and a certificate bundle that’s trusted in one browser might not be trusted in another; this can happen for several reasons:

Different browsers trust different root certificates.
Some browsers trust more root certificates than others. For example, the NSS root store used in Firefox trusts 143 certificates; however, Windows 8.1 trusts 391 certificates. So a bundle with a chain that expects the browser to trust a certificate exclusive to the Windows root store will appear as untrusted in Firefox.

Older systems might have old root stores.
Some browsers run on older systems that have not been updated to trust recently-created certificate authorities. For example, Android 2.2 and earlier don't trust the “GoDaddy Root Certificate Authority G2” because that root certificate was created after they were released.

Older systems don't support modern cryptography.
Users on Windows XP SP2 and earlier can't validate certificates signed by certain intermediates. This includes the “DigiCert SHA2 High Assurance Server CA” because the hash function SHA-2 used in this certificate was standardised after Windows XP was released.

In order to provide maximum compatibility between SSL chains and browsers, you often have to pick a different certificate chain than the one originally provided to you by your CA. This alternate chain might contain a different set of intermediates that are signed by a different root certificate. Alternate chains can be troublesome. They tend to contain a longer list of certificates than the default chain from the CA, and longer chains cause site connections to be slower. This lag is because the web server needs to send more certificates (i.e. more data) to the browser, and the browser has to spend time verifying more certificates on its end. Picking the right chain can be tricky.
CFSSL helps pick the right certificate chain, selecting for performance, security, and compatibility.

How to Pick the Best Chain
The chain of trust works by following the keys used to sign certificates, and there can be multiple chains of trust for the same keys.

In this diagram, all of the intermediates have the same public key and represent the same identity (GlobalSign's intermediate signing certificate), and they are signed by the GlobalSign root key; however, the issuance date for each chain and signature type are different.

For some outdated browsers, the current GlobalSign root certificate is not trusted, so GlobalSign used an older CA (GlobalSign nv-sa) to sign their root certificate. This allows older browsers to trust certificates issued by GlobalSign.

Each of these are valid chains for some browsers, but each has drawbacks:

CloudFlare leaf → GlobalSign SHA2 Intermediate.
This chain is trusted by any browser that trusts the GS root G2 and supports SHA2 (i.e., this chain would not be trusted by a browser on Windows XP SP2).

CloudFlare leaf → 2012 GlobalSign Intermediate → GS Root G2.
This chain is trusted by any browser that trusts GS Root G2, but does not benefit from the stronger cryptographic properties of the SHA-2 hashing algorithm.

CloudFlare leaf → 2012 GlobalSign Intermediate → GS cross-signed root.
This chain is trusted by any browser that trusts the GlobalSign nv-sa root, but uses the older (and weaker) GlobalSign nv-sa root certificate.

This last chain is the most common because it’s trusted by more browsers; however, it’s also the longest, and has weaker cryptography.

CFSSL can create either the most common or the optimal bundle, and if you need help deciding, the documentation that ships with CFSSL has tips on choosing.

If you decide to create the optimal bundle, there’s a chance it might not work in some browsers; however, CFSSL is configured to let you know specifically which browsers it will not work with. For example, it will warn the user about bundles that contains certificates signed with advanced hash functions such as SHA2; they can be problematic for certain operating systems like Windows XP SP2.

CFSSL as a Certificate Authority
CFSSL is not only a tool for bundling a certificate, it can also be used as a CA. This is possible because it covers the basic features of certificate creation including creating a private key, building a certificate signature request, and signing certificates.

You can create a brand new CA with CFSSL using the -initca option. As we saw above, this takes a JSON blob with the certificate request, and creates a new CA key and certificate:

$ cfssl gencert -initca ca_csr.json

This will return a CA certificate and CA key that is valid for signing. The CA is meant to function as an internal tool for creating certificates. CFSSL should be used behind a middle layer that processes incoming requests, and ensures they conform to policy; we use it behind the CloudFlare site as an internal service.

Here’s an example of signing a certificate with CFSSL on the command line:

$ cfssl sign -ca-key=test-key.pem -ca=test-cert.pem www.example.com example.csr

Alternatively, a POST request containing the CSR and hostname can be sent to the CFSSL service via the /api/v1/cfssl/sign endpoint. To run the service call the serve command as follows:

$ cfssl serve -address=localhost -port=8888 -ca-key=test-key.pem -ca=test-cert.pem

If you already have a CFSSL instance running (in this case on localhost, but it can be anywhere), you can automate certificate creation with the gencert command’s -remote option. For example, if CFSSL is running on localhost, running the following gives you a private key and a certificate signed by the CA:



$ cfssl gencert -remote="localhost:8888" www.example.com csr.json
https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md

cfssl gencert --help 
cfssl ls
cfssl print-defaults csr
cfssl print-defaults config

cfssl gencert -initca ca-csr.json
This gives you all 3 certificates but we want each certificate in a separate file.
cfssljson --help
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

2023/03/31 15:40:50 [INFO] generating a new CA key and certificate from CSR
2023/03/31 15:40:50 [INFO] generate received request
2023/03/31 15:40:50 [INFO] received CSR
2023/03/31 15:40:50 [INFO] generating key: rsa-2048
2023/03/31 15:40:51 [INFO] encoded CSR
2023/03/31 15:40:51 [INFO] signed certificate with serial number 595408172640289122691190423472144634592996419026
# we now have 3 files generated
ls
ca.csr  ca-key.pem  ca.pem

https://lapo.it/asn1js/
look at the certificate generated
xclip -selection clipboard -i < 'ca.pem'

openssl x509 -in ca.pem -text

cfssl gencert --help
cfssl gencert -ca cert -ca-key key [-config config] [-profile profile] [-hostname hostname] CSRJSON

cfssl gencert -ca cert -ca-key key