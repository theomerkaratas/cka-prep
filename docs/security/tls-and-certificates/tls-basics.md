- A certificate is used to guarantee trust between two parties during a transaction.
- Certificates ensure that the communication between the user and the server is encrypted and the server is who it says it is.
- **The Basic Risk:**
    - When logging into a website without a secure connection, credentials (username and password) are sent in plain text, allowing an eavesdropper ("man-in-the-middle") to intercept them.
- **Symmetric Encryption & Its Limitation:**
    - To protect the data, it must be encrypted.
    - **Symmetric encryption** uses a single shared key for both encrypting and decrypting data.
    - **The Key Exchange Problem:** The server needs this key to decrypt the data. If the key is transmitted across the same insecure network, an attacker can intercept the key as well and use it to decrypt the traffic.
- **The Solution: Asymmetric Encryption:**
    - **Asymmetric encryption** solves the key exchange dilemma by using a mathematically linked pair of keys: a **public key** and a **private key**.
    - **Private Key:** Kept secret and never shared; only the owner has access to it.
    - **Public Key:** Freely distributed and accessible to anyone.
- **How the "Lock and Key" Analogy Works:**
    - Think of the **public key** as an open padlock: anyone can use it to lock a box (encrypt data).
    - Once snapped shut, only the holder of the matching **private key** can unlock the box (decrypt the data).
    - Even if an attacker intercepts the public key or the encrypted data, they cannot decrypt it because they do not possess the private key.

### SSH Key-Based Authentication
- **Generating the Key Pair:**
	- We can create both a private and a public key using the `ssh-keygen` command.
	- This command generates two distinct files: `id_rsa`, which is our private key, and `id_rsa.pub`, which is our public key.
	- The private key must always stay safe on our local machine and never be shared, while the public key is intended to be distributed freely.
- **Securing the Server:**
	- We then secure our server by locking down access to it, allowing entry only to someone who holds the matching key pair.
	- We do this by placing our public key on the server, specifically by appending it to the server's `~/.ssh/authorized_keys` file.
	- We can always verify which keys are authorized on the server by running `cat ~/.ssh/authorized_keys`.
- **Connecting to the Server:**
	- When we want to connect via SSH, we point to our local private key using the `-i` flag, like `ssh -i id_rsa user1@server1`.
	- The server checks our private key against the public key listed in its `authorized_keys` file.
	- If the two keys match, the server verifies our identity and grants access without asking for a password.

>Usually, certificates with public key are named `.crt` or `.pem` extension. Private keys are usually with extension `.key` or `-key.pem.` For example; `server.key` and `server-key.pem`