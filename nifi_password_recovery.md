# NiFi - Password Recovery

Reference: https://medium.com/@danielyahn/nifi-password-recovery-part-1-ad2b7f25bbc3

You have a multi-tenants NiFi environment, each tenant type their passwords to connect
to various data sources, and owners of these flows leave your company, and you have know
idea what the passwords are. Sounds familiar?And, of course, these passwords are encrypted
when NiFi store them on disk.



Recently, we ran into a problem of locating all encrypted passwords while migrating flows
from one NiFi cluster to another. Luckily, NiFi provides a
[flow migration tool](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html#existing-flow-migration) to migrate
entire canvas (flow.xml.gz). However, it didn’t solve our problems completely because we
didn’t want to blindly dump the whole canvas, but wanted to use this exercise as a chance
to re-engineer existing flows. So, I took a dive into
[code repository](https://github.com/apache/nifi) of this flow migration
tool. And I quickly found out that with one line code change, we can decrypt all the store
passwords.

My colleague already had
[a very succinct post](https://www.oopsmydata.com/2018/10/decrypting-nifi-passwords-one-of-great.html)
about this, so make sure to read it. But with this post I’d like to add few more details,
and have discussion on share lessons learned from this exercise (Part 2).

## How to Decrypt Passwords in flow.xml.gz

This is a code snippet from Apache NiFi Github’s
[ConfigEncryptionTool.groovy](https://github.com/apache/nifi/blob/ea9b0db2f620526c8dd0db595cf8b44c3ef835be/nifi-toolkit/nifi-toolkit-encrypt-config/src/main/groovy/org/apache/nifi/properties/ConfigEncryptionTool.groovy)
(starts around line 98). This method is executed by the flow migration tool.

Let’s take a look at line 4 and its functional expression. For each encrypted value in
flow.xml:
1. Decrypt (line 5)
2. Encrypt (line 6,7)
3. Write the encrypted value to new flow.xml

```
private String migrateFlowXmlContent(String flowXmlContent, String existingFlowPassword, String newFlowPassword, String existingAlgorithm = DEFAULT_FLOW_ALGORITHM, String existingProvider = DEFAULT_PROVIDER, String newAlgorithm = DEFAULT_FLOW_ALGORITHM, String newProvider = DEFAULT_PROVIDER) {
    /* ... */
    // Scan the XML content and identify every encrypted element, decrypt it, and replace it with the re-encrypted value
    String migratedFlowXmlContent = flowXmlContent.replaceAll(WRAPPED_FLOW_XML_CIPHER_TEXT_REGEX) { String wrappedCipherText ->
        String plaintext = decryptFlowElement(wrappedCipherText, existingFlowPassword, existingAlgorithm, existingProvider)
        byte[] cipherBytes = encryptCipher.doFinal(plaintext.bytes)
        byte[] saltAndCipherBytes = concatByteArrays(encryptionSalt, cipherBytes)
        elementCount++
        "enc{${Hex.encodeHex(saltAndCipherBytes)}}"
    }
    /* ... */
    migratedFlowXmlContent
}
```

YAY! Decrypted clear text value is stored in String. Simply print it out! So I just simply
added a logging statement before line 4.

```
private String migrateFlowXmlContent(String flowXmlContent, String existingFlowPassword, String newFlowPassword, String existingAlgorithm = DEFAULT_FLOW_ALGORITHM, String existingProvider = DEFAULT_PROVIDER, String newAlgorithm = DEFAULT_FLOW_ALGORITHM, String newProvider = DEFAULT_PROVIDER) {
    /* ... */
    // additional logging statement that got added
    flowXmlContent.findAll(WRAPPED_FLOW_XML_CIPHER_TEXT_REGEX) {String wrappedCipherText ->
            logger.warn("Original: "+wrappedCipherText+"\t Decrypted:"+decryptFlowElement(wrappedCipherText, existingFlowPassword, existingAlgorithm, existingProvider))
        }
    
    // Scan the XML content and identify every encrypted element, decrypt it, and replace it with the re-encrypted value
    String migratedFlowXmlContent = flowXmlContent.replaceAll(WRAPPED_FLOW_XML_CIPHER_TEXT_REGEX) { String wrappedCipherText ->
        String plaintext = decryptFlowElement(wrappedCipherText, existingFlowPassword, existingAlgorithm, existingProvider)
        byte[] cipherBytes = encryptCipher.doFinal(plaintext.bytes)
        byte[] saltAndCipherBytes = concatByteArrays(encryptionSalt, cipherBytes)
        elementCount++
        "enc{${Hex.encodeHex(saltAndCipherBytes)}}"
    }
    /* ... */
    migratedFlowXmlContent
}
```

I had to spend some time on resolving dependencies to compile the jar for the tool.
I ported nifi_toolkit_encrypt subproject from [NiFi GitHub](https://github.com/apache/nifi)
(currently version 1.6.0). I checked in this subproject to my GitHub. So feel free to clone
the repo and build the jar yourself.

Now let’s try to run the tool as described in [NiFi Admin Guide](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html#encrypt-config_tool).

## Prerequisite

1. Since we don’t want to modify NiFi production scripts, copy your NiFi’s nifi-toolkit
directory to your local directory (let’s call it ```<copied nifi-toolkit location>```).
FYI, for my HDF Sandbox distribution, it’s in
```/var/lib/ambari-agent/cache/common-services/NIFI/1.0.0/package/files/nifi-toolkit-1.5.0.3.1.1.0–35/```

2. Copy ```flow.xml.gz``` and ```nifi.properties``` to ```<copied nifi-toolkit location>```
for same reason. FYI, for my HDF Sandbox distribution, it’s in

3. To replace existing jar, delete ```<copied nifi-toolkit location>/lib/nifi-toolkit-encrypt-config-<version>.jar```
and upload the compiled jar to ```<copied nifi-toolkit location>/lib/```.

## Recover Passwords

This is the command to run (assuming you are in the copied directory). Each flag is
explained in [NiFi Documentation](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html#encrypt-config_tool).

```
./bin/encrypt-config.sh -b /path/to/dest/bootstrap.conf \
   -f flow.xml.gz -g flow.xml.gz.new -n nifi.properties \
   -o nifi.properties.new -s newpassword -x
```

Running this tool, password should be printed out to the standard output like this:

```
2018/10/24 17:34:02 WARN [main] org.apache.nifi.properties.ConfigEncryptionTool:
Original: enc{5908fa1bc38de0628adea3d489b2b35f56a9637ccfea6a04bbb2b810a02020fc}
Decrypted:<decrypted value>
```

