###Secure your crenditial with Nitrokey Storage and Nitrokey HSM and keepass

keyword: Nitrokey HSM, Nitrokey Storage
The best case is local computer is always secure(no way), the tokens are
intact. So we don't do much to thinking about the fallback plan. But in
reality...
<pre>
Nitrokey HSM --------> Secure all certificate and key (for 802.1x/VPN/Web)
     |
     |
     |fallback
     +---------------> Using air gap local backup
     
                        Figure 1
</pre>

<pre>
Nitrokey Storage --->  OTP  ----> Secure Keepass(HOTP)
       |                 |                  |
       |                 |                  |fallback
       |                 |                  +--------> OTP secret
       |                 |
       |                 +-------> Secure Github(TOTP)
       |                 |                  |
       |                 |                  |fallback
       |                 |                  +--------> Secure code
       |                 |
       |                 +-------> Secure Gmail(TOTP)
       |                 |                  |
       |                 |                  |fallback
       |                 |                  +--------> Secure code, sms
       |                 |
       |                 +-------> Secure Dropbox
       |                 |                  |
       |                 |                  |fallback
       |                 |                  +--------> Secure code, sms
       |                 |                  |
       |                 |                  |backup
       |                 |                  +--------> keepass(kdbx)
       |                 |
       |                 +-------> Secure more
       |
       |
       +------------->  Storage  ---> Encrypted Storage ---> Store Keepass(kdbx)
       |
       |
       |
       |
       |
       |
       |
       |
       +------------->  OpenPGP Card ---> Store PGP key
                              |                 
                              +---------> gpg agent ---> SSH agent


                       Figure 2
</pre>
<pre>
Keepass ---> Web Service Password ---> Dropbox password
   |                   |
   |                   +-------------> Google password
   |                   |
   |                   +-------------> Github password
   |                   |
   |                   +-------------> Social account
   |                   |
   |                   +-------------> More
   |
   |
   |
   +-------> Remote Server Password


                       Figure 3

</pre>


######Sence 1

We lost our OTP token (Nitrokey Storage), So we need to use OTP secret, secure
code or sms to recovery it. See Figure 2. In this case, we should secure our
otp secret and secure code and sms carefully.

For secure code, you could store in keepass(if you think you can keep it safe.
My choice, using keepass with nitrokey stroage build-in OTP function)

For sms, I don't trust my Cell Phone Provider and the Rogue base station
attack. So I will use sms as another factor of two factor authentication and
encrypted again. For example I will using GnuPG encrypt my important email.

For keepass OTP secret, I will store it offline

######Sence 2

In case our kdbx(keepass password database) is damage or missing. We can use Dropbox
to synchronize the kdbx file and do not syncronize the key file(if you use key
file rather than OTP). Or send kdbx file as attachment to your own email with
PGP encrypted, after every change you make.

######Sence 3

If you encrypt your email and file with GnuGP, you should secure your private
key very carefully. For daily use, I will use a OpenPGP card to store my
subkeys and store my private key in air-gap environment.


#####Reference
http://security.stackexchange.com/questions/31594/what-is-a-good-general-purpose-gnupg-key-setup
