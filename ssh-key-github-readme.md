# SSH Key Generation and GitHub Integration on Ubuntu

This guide outlines the step-by-step process for generating an SSH key on an Ubuntu machine and adding it to your GitHub account. Using SSH keys provides a secure way to log into GitHub without typing your username and personal access token for every Git operation.

## 📋 Table of Contents
1. [Prerequisites](#prerequisites)
2. [Step 1: Check for Existing SSH Keys](#step-1-check-for-existing-ssh-keys)
3. [Step 2: Generate a New SSH Key](#step-2-generate-a-new-ssh-key)
4. [Step 3: Add the SSH Key to the ssh-agent](#step-3-add-the-ssh-key-to-the-ssh-agent)
5. [Step 4: Copy the Public Key](#step-4-copy-the-public-key)
6. [Step 5: Add the Key to GitHub](#step-5-add-the-key-to-github)
7. [Step 6: Test the Connection](#step-6-test-the-connection)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites
*   An Ubuntu system (Desktop or Server).
*   Access to the terminal (`Ctrl` + `Alt` + `T`).
*   An active [GitHub account](https://github.com/).
*   `git` installed on your machine (`sudo apt install git`).

---

## Step 1: Check for Existing SSH Keys
Before generating a new key, check if you already have one to avoid overwriting it.

1.  Open your terminal.
2.  List the files in your `.ssh` directory:
    ```bash
    ls -al ~/.ssh
    ```
3.  Look for files named:
    *   `id_ed25519.pub`
    *   `id_rsa.pub`
    *   `id_ecdsa.pub`

> **Note:** If you see these files and wish to use them, skip to **Step 3**. If you receive an error saying the directory doesn't exist, proceed to **Step 2**.

---

## Step 2: Generate a New SSH Key
We will use the **Ed25519** algorithm, which is the current standard for security and performance.

1.  Run the following command (replace the email with your GitHub email address):
    ```bash
    ssh-keygen -t ed25519 -C "your_email@example.com"
    ```

    *If your system is very old and doesn't support Ed25519, use RSA: `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`*

2.  **File Location:** When prompted to "Enter a file in which to save the key," press **Enter** to accept the default location (`/home/username/.ssh/id_ed25519`).

3.  **Passphrase:** You will be asked to enter a passphrase.
    *   **Recommended:** Type a secure passphrase (nothing will show on screen while typing).
    *   **Optional:** Press Enter twice for no passphrase (less secure).

---

## Step 3: Add the SSH Key to the ssh-agent
To avoid typing your passphrase every time you use Git, add your key to the SSH agent.

1.  Start the ssh-agent in the background:
    ```bash
    eval "$(ssh-agent -s)"
    ```
    *Output should look like: `Agent pid 1234`*

2.  Add your SSH private key to the ssh-agent:
    ```bash
    ssh-add ~/.ssh/id_ed25519
    ```
    *(If you created the key with a different name, replace `id_ed25519` with your filename).*

---

## Step 4: Copy the Public Key
**⚠️ Important:** You must copy the contents of the **Public** key (`.pub`), never the Private key.

1.  Display the public key content in the terminal:
    ```bash
    cat ~/.ssh/id_ed25519.pub
    ```
2.  Select and copy the entire output (starting with `ssh-ed25519` and ending with your email).

    *Alternatively, if you have `xclip` installed:*
    ```bash
    xclip -selection clipboard < ~/.ssh/id_ed25519.pub
    ```

---

## Step 5: Add the Key to GitHub

1.  Log in to [GitHub](https://github.com/).
2.  Click your profile photo in the top-right corner and select **Settings**.
3.  In the user settings sidebar, click **SSH and GPG keys**.
4.  Click the green button **New SSH key** (or "Add SSH key").
5.  **Title:** Give it a descriptive name (e.g., "Ubuntu Work Laptop").
6.  **Key type:** Leave it as "Authentication Key".
7.  **Key:** Paste the public key you copied in Step 4.
8.  Click **Add SSH key**.
9.  You may be asked to confirm your GitHub password or use 2FA.

---

## Step 6: Test the Connection
Verify that your system can communicate with GitHub via SSH.

1.  Run the following command:
    ```bash
    ssh -T git@github.com
    ```
2.  You may see a warning like:
    > The authenticity of host 'github.com...' can't be established.
    > Are you sure you want to continue connecting (yes/no/[fingerprint])?

3.  Type `yes` and press **Enter**.

4.  **Success Message:**
    > Hi username! You've successfully authenticated, but GitHub does not provide shell access.

---

## Switching Remote URLs (Optional)
If you previously cloned a repository using HTTPS, you need to switch the remote URL to SSH to use your new key.

1.  Go to your repository folder:
    ```bash
    cd /path/to/your/repo
    ```
2.  Check current remote:
    ```bash
    git remote -v
    ```
3.  Set the new SSH URL:
    ```bash
    git remote set-url origin git@github.com:USERNAME/REPOSITORY.git
    ```

---

## Troubleshooting

### "Permission denied (publickey)"
*   Ensure you copied the key from the `.pub` file, not the private file.
*   Ensure the key is added to the agent (`ssh-add -l`).
*   Ensure the public key is correctly pasted in GitHub settings without newlines or extra spaces.

### "Agent admitted failure to sign using the key"
*   This usually happens if the agent isn't running properly. Try logging out and back into Ubuntu, or re-run the commands in Step 3.

### Key file permissions are too open
*   If you get a warning about "UNPROTECTED PRIVATE KEY FILE", run:
    ```bash
    chmod 600 ~/.ssh/id_ed25519
    ```