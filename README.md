# pyiCloud

[![Build Status](https://travis-ci.org/picklepete/pyicloud.svg?branch=master)](https://travis-ci.org/picklepete/pyicloud)
[![Library version](https://img.shields.io/pypi/v/pyicloud.svg)](https://pypi.org/project/pyicloud)
[![Supported versions](https://img.shields.io/pypi/pyversions/pyicloud.svg)](https://pypi.org/project/pyicloud)
[![Downloads](https://pepy.tech/badge/pyicloud)](https://pypi.org/project/pyicloud)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/picklepete/pyicloud)

PyiCloud is a module which allows pythonistas to interact with iCloud webservices. It's powered by the fantastic [requests](https://github.com/kennethreitz/requests) HTTP library.

At its core, PyiCloud connects to iCloud using your username and password, then performs calendar and iPhone queries against their API.

> **Linux users:** pyicloud fails out-of-the-box on Linux due to Apple returning HTTP 421 on the session validation endpoint. This fork includes a fix. See [Known Issues and Fixes](#known-issues-and-fixes) for the explanation and [Mounting iCloud Drive on Linux](#mounting-icloud-drive-on-linux) for a full setup guide.

---

## Known Issues and Fixes

### HTTP 421 "Misdirected Request" on Linux (non-Apple networks)

When using pyicloud from Linux or any non-Apple network, the initial call to `setup.icloud.com/validate` may return HTTP 421 (Misdirected Request). This is caused by Apple's HTTP/2 connection coalescing behavior, where the server routes the TCP connection to a different virtual host than expected. It is **not a real authentication failure**.

Prior to this fix, pyicloud would treat 421 as an invalid session token and attempt a full re-login with username and password. This re-login then hit `idmsa.apple.com` which returns 503 (rate-limiting), causing a `PyiCloudFailedLoginException` even when the session was perfectly valid.

**The fix** (in `authenticate()`): when 421 is returned from `/validate`, fall through to `_authenticate_with_token()` which uses the existing `session_token` and `trust_token` to call `/accountLogin` directly. This succeeds without requiring a password or 2FA, and without triggering Apple's rate-limiting on the auth endpoint.

Confirmed working on:

- Arch Linux (kernel 6.18, Python 3.14)
- pyicloud 1.0.0 + iCloudDriveFuse FUSE driver

---

## Authentication

Authentication without using a saved password is as simple as passing your username and password to the `PyiCloudService` class:

```python
from pyicloud import PyiCloudService
api = PyiCloudService('jappleseed@apple.com', 'password')
```

In the event that the username/password combination is invalid, a `PyiCloudFailedLoginException` exception is thrown.

If the country/region setting of your Apple ID is China mainland, pass `china_mainland=True`:

```python
from pyicloud import PyiCloudService
api = PyiCloudService('jappleseed@apple.com', 'password', china_mainland=True)
```

You can also store your password in the system keyring using the command-line tool:

```console
$ icloud --username=jappleseed@apple.com
ICloud Password for jappleseed@apple.com:
Save password in keyring? (y/N)
```

If you would like to delete a stored password:

```console
$ icloud --username=jappleseed@apple.com --delete-from-keyring
```

**Note:** Authentication will expire after an interval set by Apple, currently two months.

### Two-step and two-factor authentication (2SA/2FA)

```python
if api.requires_2fa:
    print("Two-factor authentication required.")
    code = input("Enter the code you received of one of your approved devices: ")
    result = api.validate_2fa_code(code)
    print("Code validation result: %s" % result)

    if not result:
        print("Failed to verify security code")
        sys.exit(1)

    if not api.is_trusted_session:
        print("Session is not trusted. Requesting trust...")
        result = api.trust_session()
        print("Session trust result %s" % result)

        if not result:
            print("Failed to request trust. You will likely be prompted for the code again in the coming weeks")
elif api.requires_2sa:
    import click
    print("Two-step authentication required. Your trusted devices are:")

    devices = api.trusted_devices
    for i, device in enumerate(devices):
        print(
            "  %s: %s" % (i, device.get('deviceName',
            "SMS to %s" % device.get('phoneNumber')))
        )

    device = click.prompt('Which device would you like to use?', default=0)
    device = devices[device]
    if not api.send_verification_code(device):
        print("Failed to send verification code")
        sys.exit(1)

    code = click.prompt('Please enter validation code')
    if not api.validate_verification_code(device, code):
        print("Failed to verify verification code")
        sys.exit(1)
```

---

## Devices

```python
>>> api.devices
{
'i9vbKRGIcLYqJnXMd1b257kUWnoyEBcEh6yM+IfmiMLh7BmOpALS+w==': <AppleDevice(iPhone 4S: Johnny Appleseed's iPhone)>,
'reGYDh9XwqNWTGIhNBuEwP1ds0F/Lg5t/fxNbI4V939hhXawByErk+HYVNSUzmWV': <AppleDevice(MacBook Air 11": Johnny Appleseed's MacBook Air)>
}

>>> api.devices[0]
<AppleDevice(iPhone 4S: Johnny Appleseed's iPhone)>
>>> api.iphone
<AppleDevice(iPhone 4S: Johnny Appleseed's iPhone)>
```

---

## Find My iPhone

### Location

```python
>>> api.iphone.location()
{'timeStamp': 1357753796553, 'locationFinished': True, 'longitude': -0.14189, 'positionType': 'GPS', 'locationType': None, 'latitude': 51.501364, 'isOld': False, 'horizontalAccuracy': 5.0}
```

### Status

```python
>>> api.iphone.status()
{'deviceDisplayName': 'iPhone 5', 'deviceStatus': '200', 'batteryLevel': 0.6166913, 'name': "Peter's iPhone"}
```

### Play Sound

```python
api.iphone.play_sound()
```

### Lost Mode

```python
phone_number = '555-373-383'
message = 'Thief! Return my phone immediately.'
api.iphone.lost_device(phone_number, message)
```

---

## Calendar

```python
api.calendar.events()

from_dt = datetime(2012, 1, 1)
to_dt = datetime(2012, 1, 31)
api.calendar.events(from_dt, to_dt)

api.calendar.get_event_detail('CALENDAR', 'EVENT_ID')
```

---

## Contacts

```python
>>> for c in api.contacts.all():
>>>     print(c.get('firstName'), c.get('phones'))
John [{'field': '+1 555-55-5555-5', 'label': 'MOBILE'}]
```

---

## File Storage (Ubiquity)

```python
>>> api.files.dir()
['.do-not-delete', '.localized', 'com~apple~Notes', ...]

>>> api.files['com~apple~Notes']['Documents']['Some Document'].open().content
'Hello, these are the file contents'
```

---

## File Storage (iCloud Drive)

```python
>>> api.drive.dir()
['Holiday Photos', 'Work Files']

>>> drive_file = api.drive['Holiday Photos']['2013']['Sicily']['DSC08116.JPG']
>>> drive_file.name
'DSC08116.JPG'
```

Download a file:

```python
from shutil import copyfileobj
with drive_file.open(stream=True) as response:
    with open(drive_file.name, 'wb') as file_out:
        copyfileobj(response.raw, file_out)
```

Create, rename, delete, upload:

```python
api.drive['Holiday Photos'].mkdir('2020')
api.drive['Holiday Photos']['2020'].rename('2020_copy')
api.drive['Holiday Photos']['2020_copy'].delete()

with open('Vacation.jpeg', 'rb') as file_in:
    api.drive['Holiday Photos'].upload(file_in)
```

---

## Photo Library

```python
>>> api.photos.all
<PhotoAlbum: 'All Photos'>

>>> api.photos.albums['Screenshots']
<PhotoAlbum: 'Screenshots'>

photo = next(iter(api.photos.albums['Screenshots']), None)
download = photo.download()
with open(photo.filename, 'wb') as opened_file:
    opened_file.write(download.raw.read())

download = photo.download('thumb')
with open(photo.versions['thumb']['filename'], 'wb') as thumb_file:
    thumb_file.write(download.raw.read())
```

---

## Mounting iCloud Drive on Linux

You can mount iCloud Drive as a local filesystem on Linux using [iCloudDriveFuse](https://github.com/ixs/iCloudDriveFuse).

### Prerequisites

```bash
# Arch Linux
sudo pacman -S fuse3 python-cachetools
yay -S python-pyicloud-git python-fusepy

# Debian/Ubuntu
sudo apt install fuse3 python3-cachetools
pip install pyicloud fusepy
```

### Step 1 - Authenticate and save session cookies

Run the `icloud` CLI tool once to authenticate and persist the session:

```bash
icloud --username your@apple.id
```

Enter your password when prompted. If 2FA is enabled, approve the push on your Apple device and enter the 6-digit code.

Move the session cookies to a persistent location so they survive reboots:

```bash
mkdir -p ~/.config/pyicloud
chmod 700 ~/.config/pyicloud
cp /tmp/pyicloud/$(whoami)/* ~/.config/pyicloud/
```

### Step 2 - Store credentials in .netrc

```bash
touch ~/.netrc && chmod 600 ~/.netrc
```

Add to `~/.netrc`:

```
machine icloud
login your@apple.id
password yourpassword
```

### Step 3 - Clone iCloudDriveFuse and patch cookie directory

```bash
git clone https://github.com/ixs/iCloudDriveFuse.git ~/iCloudDriveFuse
```

Edit `~/iCloudDriveFuse/iCloudDriveFuse.py` and update the `__init__` method:

```python
def __init__(self):
    self.username, _, self.password = netrc.netrc().authenticators("icloud")
    cookie_dir = os.path.expanduser("~/.config/pyicloud")
    self._api = PyiCloudService(self.username, self.password, cookie_directory=cookie_dir)
```

### Step 4 - Mount

```bash
mkdir -p ~/iCloud
python ~/iCloudDriveFuse/iCloudDriveFuse.py ~/iCloud
```

### Step 5 - Auto-mount via systemd (optional)

Create `~/.config/systemd/user/icloud-drive.service`:

```ini
[Unit]
Description=iCloud Drive FUSE Mount
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/python /home/YOUR_USER/iCloudDriveFuse/iCloudDriveFuse.py /home/YOUR_USER/iCloud
ExecStop=/usr/bin/fusermount -u /home/YOUR_USER/iCloud
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

Enable it:

```bash
systemctl --user daemon-reload
systemctl --user enable --now icloud-drive.service
```

### Maintenance

Apple trust tokens expire roughly every two months. When the mount stops working, re-authenticate:

```bash
icloud --username your@apple.id
cp /tmp/pyicloud/$(whoami)/* ~/.config/pyicloud/
systemctl --user restart icloud-drive.service
```

### Note on HTTP 421 errors

If you see `Authentication required for Account. (421)` in the logs, see [Known Issues and Fixes](#known-issues-and-fixes) above. Make sure you are using a version of pyicloud that includes the 421 fix in `authenticate()`. The fix is included in this fork.

---

## Code samples

See the [code samples file](CODE_SAMPLES.md).
