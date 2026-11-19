# Lab: Publish a Web Page as a Tor Onion Service

## Goal

Run a small web server on your computer, publish it as a Tor onion service, and open the page from a **different device** through Tor (for example, from your phone).

You will learn how a local TCP service can be reached through Tor without a public IP address or router port forwarding, how Tor uses a `torrc` configuration file, and the difference between running Tor directly and running it as an operating-system service.

You may work on **Windows, Linux, or macOS**. On the Windows lab computers, you do **not** have administrator privileges; the instructions below do not require them.

## 1. Start a local web server

Choose the section for your operating system. Keep the web server running while you complete the lab.

### Windows (lab computers): portable Caddy

1. Download the Windows executable of [Caddy](https://caddyserver.com/download). It is a standalone web server; you do not need to install a Windows service or obtain administrator privileges.
2. Open **Command Prompt** (`cmd.exe`) and create a working directory:

   ```cmd
   mkdir "%USERPROFILE%\tor-lab"
   cd /d "%USERPROFILE%\tor-lab"
   mkdir site
   ```

3. Place `caddy.exe` in the `tor-lab` directory.
4. Create `site\index.html` using Notepad. For example:

   ```cmd
   notepad site\index.html
   ```

   Write a short page that identifies your web server, and save it.
5. Start the web server:

   ```cmd
   caddy.exe file-server --listen 127.0.0.1:8000 --root site
   ```

6. Open `http://127.0.0.1:8000/` in your normal browser. You should see your page.

**Alternative:** You may use [MiniWeb](https://sourceforge.net/projects/miniweb/files/miniweb/0.8/miniweb-win32-20130309.zip/download) instead. Start `miniweb.exe` and check which address and port it actually listens on. The target in `HiddenServicePort` (in section 2) must match the web server's reachable address and port. A web server bound to `127.0.0.1` is preferable for this exercise.

### Linux or macOS: Python HTTP server

Open a terminal:

```sh
mkdir -p "$HOME/tor-lab/site"
cd "$HOME/tor-lab"
printf 'Hello from %s\n' "$USER" > site/index.html
python3 -m http.server 8000 --bind 127.0.0.1 --directory site
```

You may replace `site/index.html` with your own HTML page. Open `http://127.0.0.1:8000/` in your normal browser and check that it works.

If `python3` is not installed, install Python 3 using your package manager (or use another HTTP server that can listen on `127.0.0.1:8000`).

## 2. Configure Tor

For the first run, **run Tor manually using your own `torrc`**. This also works on computers where you cannot manage system services.

### Windows: download portable Tor

1. Download the **Tor Expert Bundle for Windows** from [the Tor Project](https://www.torproject.org/download/tor/). This is the Tor program, not Tor Browser.
2. Extract the bundle into `%USERPROFILE%\tor-lab`. Depending on the archive format, you may need to extract it twice. Arrange the files so that the directory contains `tor\tor.exe` (and the bundle's other files).
3. In the `tor-lab` directory, create a plain-text file named **`torrc`** (not `torrc.txt`):

   ```cmd
   cd /d "%USERPROFILE%\tor-lab"
   notepad torrc
   ```

4. Put these lines in the file:

   ```text
   SocksPort 0
   DataDirectory ./tor-data
   HiddenServiceDir ./onion-service
   HiddenServicePort 80 127.0.0.1:8000
   ```

5. In a **second Command Prompt window**, from the `tor-lab` directory, check the configuration and run Tor:

   ```cmd
   cd /d "%USERPROFILE%\tor-lab"
   tor\tor.exe --verify-config -f torrc
   tor\tor.exe -f torrc
   ```

Keep this window open. Wait until Tor reports `Bootstrapped 100% (done)`.

### Linux or macOS: run Tor manually

Install the `tor` command if you do not already have it. For example:

**Debian/Ubuntu:**

```sh
sudo apt install tor
```

**Gentoo:**

```sh
sudo emerge --ask net-vpn/tor
```

**macOS with Homebrew:**

```sh
brew install tor
```

If you cannot install packages, you can instead obtain the appropriate [Tor Expert Bundle](https://www.torproject.org/download/tor/) and run its Tor executable from your home directory. Use the executable's actual path in place of `tor` below.

Create a configuration file in your own working directory:

```sh
cd "$HOME/tor-lab"
cat > torrc <<'END'
SocksPort 0
DataDirectory ./tor-data
HiddenServiceDir ./onion-service
HiddenServicePort 80 127.0.0.1:8000
END
```

Start Tor **from this directory**:

```sh
tor --verify-config -f ./torrc
tor -f ./torrc
```

Wait for `Bootstrapped 100% (done)`. Keep this terminal open. If a system Tor service is already running, these settings keep this instance's data and onion-service identity separate from it.

### What does `torrc` mean?

- `SocksPort 0` — do not open a local SOCKS proxy port; this Tor instance is being used to host a service.
- `DataDirectory ./tor-data` — store this Tor instance's working data in the lab directory.
- `HiddenServiceDir ./onion-service` — store the onion service's address and **private keys** here.
- `HiddenServicePort 80 127.0.0.1:8000` — visitors use port **80** on the onion address; Tor forwards the connection to your local web server on port **8000**.

The relative directory paths above are interpreted relative to the directory **from which Tor is started**, so the `cd` commands matter. Tor normally creates the data and onion-service directories itself.

You do **not** need a `ControlPort`, a Windows service, a public IP address, or an incoming firewall rule for this lab.

## 3. Find the onion address

Once Tor has started, read the `hostname` file it generated.

**Windows (in a new Command Prompt):**

```cmd
cd /d "%USERPROFILE%\tor-lab"
type onion-service\hostname
```

**Linux/macOS:**

```sh
cat "$HOME/tor-lab/onion-service/hostname"
```

The result is a long address ending in `.onion` (a version 3 onion-service address). **Share the address, not the private-key files** in `onion-service`.

## 4. Open the page from another device

1. On an **Android** phone, use [Tor Browser for Android](https://www.torproject.org/download/#android). On **iOS**, use [Onion Browser](https://onionbrowser.com/). Alternatively, use Tor Browser on a separate computer.
2. Connect the browser to Tor.
3. Enter your address in this form:

   ```text
   http://YOUR_ADDRESS.onion/
   ```

4. Check that you can see the same page that you served locally. For an extra test, try your phone's cellular connection rather than the lab Wi-Fi.

Your browser on the **host computer** does not have to use Tor to display `http://127.0.0.1:8000/`. The **remote device** must use Tor to reach the `.onion` address.

## 5. Another way to run Tor: as a service

A background system service is useful for a long-running onion service. **This is an alternative to the manual Tor process in section 2, not something to run simultaneously with that process using the same configuration and data.** Stop your manually started Tor with `Ctrl+C` before trying this option.

A system service typically runs as a dedicated account, not as your interactive user. It also reads its **own** `torrc`. Therefore, the service needs a directory where its account can write the onion-service keys.

### Linux with systemd (Debian/Ubuntu example)

Edit `/etc/tor/torrc` with administrator privileges and add:

```text
HiddenServiceDir /var/lib/tor/lab-onion
HiddenServicePort 80 127.0.0.1:8000
```

Then restart the packaged Tor instance and read its onion address:

```sh
sudo systemctl restart tor@default
sudo systemctl status tor@default
sudo cat /var/lib/tor/lab-onion/hostname
```

The service name varies by distribution; for example, some use `tor.service`. You can inspect the installed units with `systemctl list-unit-files 'tor*'`.

### Linux with OpenRC (Gentoo example)

Add the same two lines to `/etc/tor/torrc`, then run:

```sh
sudo rc-service tor restart
sudo rc-service tor status
sudo cat /var/lib/tor/lab-onion/hostname
```

### macOS with Homebrew

Homebrew can run Tor as a **per-user service** without `sudo`. Find its configuration directory:

```sh
brew --prefix
```

Edit `$(brew --prefix)/etc/tor/torrc` and add the following, replacing `YOUR_USERNAME` with your actual macOS username:

```text
HiddenServiceDir /Users/YOUR_USERNAME/tor-lab/service-onion
HiddenServicePort 80 127.0.0.1:8000
```

Start or restart the service:

```sh
brew services restart tor
brew services list
cat "$HOME/tor-lab/service-onion/hostname"
```

The service may generate a **different** `.onion` address from the manually started Tor instance because its `HiddenServiceDir` is different. Try that address from your phone too.

### Windows lab computers

Registering a Windows service requires administrator privileges, which you do **not** have on the lab computers. Use the **manual** `tor.exe -f torrc` method. You should still understand the difference between a foreground process and a managed system service.

## 6. Troubleshooting

- **Local page fails:** First check `http://127.0.0.1:8000/`. Is the HTTP server still running? Is it listening on the address and port you specified?
- **Tor does not start:** Check the error text, the path to `torrc`, and directory permissions. Run `tor --verify-config -f torrc` (or `tor\tor.exe` on Windows).
- **Tor cannot bootstrap:** Check internet access and whether the network blocks Tor connections. Try a different network or consult the instructor.
- **No `hostname` file:** Check that the `HiddenServiceDir` is correct and that Tor can create it. The manual configuration's relative paths depend on your working directory.
- **Onion page fails but localhost works:** Make sure Tor has reached 100%, both processes are still running, and the `HiddenServicePort` target matches the HTTP server. The onion service may need a short time to become reachable.
- **The onion URL changed:** A different or deleted `HiddenServiceDir` produces a different onion-service identity. Manual and service configurations above use different directories.
- **The page works on your PC but not from the phone:** Check that the phone browser is actually connected to Tor; a normal browser cannot open `.onion` addresses.

## 7. Submission

Send Ashot and Norayr your screenshots and short written answers. The submission should demonstrate:

1. Your local page at `http://127.0.0.1:8000/`.
2. Your `torrc` file (show the four settings, not any private keys).
3. Tor running successfully, including its bootstrap completion.
4. The generated `.onion` hostname.
5. The same page opened from a **different device** using Tor.
6. If you tried the service method, its status and a successful visit to its onion address.

Answer these questions:

1. Why does `HiddenServicePort` mention both port `80` and port `8000`?
2. What is the difference between `127.0.0.1` and your `.onion` address?
3. Why can the phone open the page even though the host computer has no publicly reachable incoming port?
4. What changes when Tor runs as a service rather than as a process in your terminal?
5. What is stored in `HiddenServiceDir`? Which files must remain private?

**Cleanup:** Stop the HTTP server and manually launched Tor with `Ctrl+C`. If you started a Tor service only for this lab, stop it using the corresponding service manager. Remove any downloaded programs and test files from shared Windows lab computers when finished. Do not submit onion-service private keys.

## References

- [Tor Project: Set up your onion service](https://community.torproject.org/onion-services/setup/)
- [Tor Expert Bundle downloads](https://www.torproject.org/download/tor/)
- [Tor configuration manual](https://manpages.debian.org/unstable/tor/torrc.5.en.html)
- [Caddy command-line documentation](https://caddyserver.com/docs/command-line)
- [Homebrew Tor formula](https://formulae.brew.sh/formula/tor)
