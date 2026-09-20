# Vehicle Telematics MQTT

Mosquitto MQTT broker with mutual TLS authentication for firmware devices and a backend application.

## 1. Requirements

- Docker with Docker Compose
- OpenSSL 3.x
- curl for downloading a public broker trust anchor
- Certbot only when using a public Let's Encrypt certificate

Create the directories used by the commands below:

```sh
mkdir -p certs/server pki/client-ca pki/clients
chmod 700 pki/client-ca pki/clients
```

## 2. Create the private client CA

This CA authenticates firmware devices and the backend to Mosquitto. Its private key must remain on a trusted provisioning computer and must not be copied to the public MQTT server.

Generate the private CA key:

```sh
openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:3072 \
  -out pki/client-ca/ca.key

chmod 600 pki/client-ca/ca.key
```

Generate the public CA certificate:

```sh
openssl req \
  -x509 -new -sha256 -days 3650 \
  -key pki/client-ca/ca.key \
  -out certs/client-ca.crt \
  -subj '/CN=Vehicle Telematics Client CA' \
  -addext 'basicConstraints=critical,CA:TRUE' \
  -addext 'keyUsage=critical,keyCertSign,cRLSign' \
  -addext 'subjectKeyIdentifier=hash'
```

The public MQTT server receives `certs/client-ca.crt`, but never `pki/client-ca/ca.key`.

## 3. Create client certificates

Every physical device gets a unique certificate and private key. The certificate Common Name is the identity used by the Mosquitto ACL.

### Device certificate

This example creates `device-001`:

```sh
openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:2048 \
  -out pki/clients/device-001.key

openssl req \
  -new -sha256 \
  -key pki/clients/device-001.key \
  -out pki/clients/device-001.csr \
  -subj '/CN=device-001' \
  -addext 'basicConstraints=critical,CA:FALSE' \
  -addext 'keyUsage=critical,digitalSignature,keyEncipherment' \
  -addext 'extendedKeyUsage=clientAuth'

openssl x509 \
  -req -sha256 -days 365 \
  -in pki/clients/device-001.csr \
  -CA certs/client-ca.crt \
  -CAkey pki/client-ca/ca.key \
  -CAserial pki/client-ca/ca.srl \
  -CAcreateserial \
  -copy_extensions copy \
  -out pki/clients/device-001.crt

rm pki/clients/device-001.csr
chmod 600 pki/clients/device-001.key
```

Repeat these commands with a different Common Name and filenames for every device.

### Backend certificate

The ACL expects the backend identity `telemetry-backend`:

```sh
openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:2048 \
  -out pki/clients/telemetry-backend.key

openssl req \
  -new -sha256 \
  -key pki/clients/telemetry-backend.key \
  -out pki/clients/telemetry-backend.csr \
  -subj '/CN=telemetry-backend' \
  -addext 'basicConstraints=critical,CA:FALSE' \
  -addext 'keyUsage=critical,digitalSignature,keyEncipherment' \
  -addext 'extendedKeyUsage=clientAuth'

openssl x509 \
  -req -sha256 -days 365 \
  -in pki/clients/telemetry-backend.csr \
  -CA certs/client-ca.crt \
  -CAkey pki/client-ca/ca.key \
  -CAserial pki/client-ca/ca.srl \
  -CAcreateserial \
  -copy_extensions copy \
  -out pki/clients/telemetry-backend.crt

rm pki/clients/telemetry-backend.csr
chmod 600 pki/clients/telemetry-backend.key
```

## 4. Choose a broker certificate

Mosquitto needs `certs/server/fullchain.pem` and `certs/server/privkey.pem`. Choose one of the following options.

For local testing, all files can remain on the same computer. For a public deployment, create the client CA and client certificates on a trusted provisioning computer. Copy only `certs/client-ca.crt` to the MQTT server; do not copy `pki/client-ca/ca.key` or client private keys to it.

### Option A: local development certificate

This certificate is valid for `localhost`, `127.0.0.1`, and the Docker service name `mosquitto`:

```sh
openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:2048 \
  -out certs/server/privkey.pem

openssl req \
  -new -sha256 \
  -key certs/server/privkey.pem \
  -out pki/client-ca/local-broker.csr \
  -subj '/CN=localhost' \
  -addext 'basicConstraints=critical,CA:FALSE' \
  -addext 'keyUsage=critical,digitalSignature,keyEncipherment' \
  -addext 'extendedKeyUsage=serverAuth' \
  -addext 'subjectAltName=DNS:localhost,DNS:mosquitto,IP:127.0.0.1'

openssl x509 \
  -req -sha256 -days 365 \
  -in pki/client-ca/local-broker.csr \
  -CA certs/client-ca.crt \
  -CAkey pki/client-ca/ca.key \
  -CAserial pki/client-ca/ca.srl \
  -CAcreateserial \
  -copy_extensions copy \
  -out certs/server/fullchain.pem

rm pki/client-ca/local-broker.csr
```

### Option B: Let's Encrypt domain certificate

Create an `A` or `AAAA` DNS record pointing `mqtt.example.com` to the public server. Allow inbound TCP port `80` temporarily for the ACME challenge, then run on the public server:

```sh
sudo certbot certonly \
  --standalone \
  --key-type rsa \
  --rsa-key-size 2048 \
  --preferred-chain "ISRG Root X1" \
  -d mqtt.example.com
```

The RSA key and X1-compatible chain are selected for embedded-modem compatibility. Certbot otherwise defaults to an ECDSA subscriber key for new certificates.

Copy the issued files into the paths used by Docker:

```sh
sudo install \
  -m 0644 \
  /etc/letsencrypt/live/mqtt.example.com/fullchain.pem \
  certs/server/fullchain.pem

sudo install \
  -m 0600 \
  /etc/letsencrypt/live/mqtt.example.com/privkey.pem \
  certs/server/privkey.pem
```

### Option C: Let's Encrypt public IP certificate

Let's Encrypt IP certificates require Certbot 5.4 or newer and are valid for approximately six days, so automatic renewal is mandatory for anything beyond a short test. See the [official Certbot IP-certificate instructions](https://letsencrypt.org/2026/03/11/shorter-certs-certbot). Allow inbound TCP port `80`, then run:

```sh
sudo certbot certonly \
  --preferred-profile shortlived \
  --standalone \
  --key-type rsa \
  --rsa-key-size 2048 \
  --preferred-chain "ISRG Root X1" \
  --ip-address YOUR_PUBLIC_IP
```

Replace `YOUR_PUBLIC_IP` with the server's public IP, then copy the certificate:

```sh
sudo install \
  -m 0644 \
  /etc/letsencrypt/live/YOUR_PUBLIC_IP/fullchain.pem \
  certs/server/fullchain.pem

sudo install \
  -m 0600 \
  /etc/letsencrypt/live/YOUR_PUBLIC_IP/privkey.pem \
  certs/server/privkey.pem
```

## 5. Set certificate permissions

Docker mounts the certificate files from the host without changing their Linux permissions. Mosquitto runs as a non-root user, so that user must be allowed to read `privkey.pem`. Otherwise, the broker exits with `Unable to load server key file: Permission denied`.

The CA certificate and server certificate are public information. Make them readable:

```sh
sudo chmod 0644 \
  certs/client-ca.crt \
  certs/server/fullchain.pem
```

The server private key is secret, so only its owner may read it:

```sh
sudo chmod 0600 certs/server/privkey.pem
```

Use a temporary container to assign the key to the `mosquitto` account from the same image:

```sh
docker run --rm \
  --mount type=bind,source="$PWD/certs/server/privkey.pem",target=/broker.key \
  --entrypoint chown \
  eclipse-mosquitto:2.1.2-alpine \
  mosquitto:mosquitto /broker.key
```

This command does not start the broker. It only applies the correct file owner using Docker's own user mapping, then removes the temporary container.

## 6. Start Docker

Create the local environment file:

```sh
cp .env.example .env
```

For local use, keep:

```dotenv
MQTT_BIND_ADDRESS=127.0.0.1
```

For public use, change `.env` to:

```dotenv
MQTT_BIND_ADDRESS=0.0.0.0
```

Start the broker:

```sh
docker compose up -d --build mosquitto
```

For public use, allow inbound TCP port `8883` in both the cloud firewall and host firewall. Do not expose unencrypted MQTT port `1883`.

Inspect the broker:

```sh
docker compose ps
docker compose logs -f --since=1s mosquitto
```

Stop it without deleting retained messages:

```sh
docker compose down
```

Delete all broker persistence, including retained messages:

```sh
docker compose down -v
```

## 7. Client settings

All clients use:

```text
Port:             8883
MQTT version:     3.1.1
TLS verification: required
Client certificate authentication: required
```

### Prepare the firmware certificate package

Create the firmware certificate package for `device-001`:

```sh
mkdir -p pki/firmware/device-001
chmod 700 pki/firmware/device-001
```

For a locally generated broker certificate from option A, use the private
deployment CA as the broker trust anchor:

```sh
install -m 0644 \
  certs/client-ca.crt \
  pki/firmware/device-001/broker-ca.pem
```

For a Let's Encrypt broker certificate using the X1-compatible chain from
option B or C, download the ISRG Root X1 trust anchor instead:

```sh
curl -fsSL \
  https://letsencrypt.org/certs/isrgrootx1.pem \
  -o pki/firmware/device-001/broker-ca.pem

chmod 0644 pki/firmware/device-001/broker-ca.pem
```

Install the device certificate and its unencrypted private key using the names
expected by the firmware:

```sh
install -m 0644 \
  pki/clients/device-001.crt \
  pki/firmware/device-001/device.pem

install -m 0600 \
  pki/clients/device-001.key \
  pki/firmware/device-001/device-key.pem
```

The resulting device package is:

```text
pki/firmware/device-001/broker-ca.pem
pki/firmware/device-001/device.pem
pki/firmware/device-001/device-key.pem
```

`broker-ca.pem` verifies the MQTT broker. `device.pem` identifies the device,
and its Common Name must match the firmware device ID (`device-001` in this
example). `device-key.pem` is secret and must remain with the provisioned
device. Never copy the client CA private key or broker private key into this
package.

The backend receives:

```text
pki/clients/telemetry-backend.crt
pki/clients/telemetry-backend.key
```

For the locally generated broker certificate, clients also trust:

```text
certs/client-ca.crt
```

For a Let's Encrypt broker certificate, desktop and backend clients can use
their operating-system/public CA trust store. Embedded firmware without a
public CA store uses the explicitly provisioned `broker-ca.pem` above. All
clients still present their private device or backend client certificate.

Never distribute these keys:

```text
pki/client-ca/ca.key
certs/server/privkey.pem
```

## 8. MQTT topics

| Topic | Publisher | QoS | Retained |
|---|---|---:|---:|
| `v1/devices/{device_id}/telemetry` | Device | 0 | No |
| `v1/devices/{device_id}/state` | Device or Last Will | 1 | Yes |
| `v1/devices/{device_id}/config/desired` | Backend | 1 | Yes |
| `v1/devices/{device_id}/config/reported` | Device | 1 | Yes |

The backend may subscribe to:

```text
v1/devices/+/telemetry
v1/devices/+/state
v1/devices/+/config/reported
```

## 9. Test locally without repository scripts

Subscribe as the backend:

```sh
docker compose --profile tools run --rm mqtt-client \
  mosquitto_sub \
  -h mosquitto -p 8883 -V mqttv311 \
  --cafile /certs/client-ca.crt \
  --cert /clients/telemetry-backend.crt \
  --key /clients/telemetry-backend.key \
  -t 'v1/devices/+/telemetry' -v
```

In another terminal, publish as `device-001`:

```sh
docker compose --profile tools run --rm mqtt-client \
  mosquitto_pub \
  -h mosquitto -p 8883 -V mqttv311 \
  --cafile /certs/client-ca.crt \
  --cert /clients/device-001.crt \
  --key /clients/device-001.key \
  -t v1/devices/device-001/telemetry \
  -m '{"schema_version":1,"sequence":1,"speed_kph":42}' \
  -q 0
```

Publish retained configuration as the backend:

```sh
docker compose --profile tools run --rm mqtt-client \
  mosquitto_pub \
  -h mosquitto -p 8883 -V mqttv311 \
  --cafile /certs/client-ca.crt \
  --cert /clients/telemetry-backend.crt \
  --key /clients/telemetry-backend.key \
  -t v1/devices/device-001/config/desired \
  -m '{"schema_version":1,"revision":1,"configuration":{"telemetry_interval_seconds":10}}' \
  -q 1 -r
```

Clear that retained configuration:

```sh
docker compose --profile tools run --rm mqtt-client \
  mosquitto_pub \
  -h mosquitto -p 8883 -V mqttv311 \
  --cafile /certs/client-ca.crt \
  --cert /clients/telemetry-backend.crt \
  --key /clients/telemetry-backend.key \
  -t v1/devices/device-001/config/desired \
  -n -q 1 -r
```

## 10. Renew Let's Encrypt certificates

Renew with Certbot:

```sh
sudo certbot renew
```

After renewal, copy the new `fullchain.pem` and `privkey.pem` using the same `install` commands from the relevant Let's Encrypt section. Repeat the private-key permission and ownership commands from section 5, then recreate Mosquitto:

```sh
docker compose up -d --force-recreate mosquitto
```

Automate those renewal, copy, and recreate steps before relying on a Let's Encrypt IP certificate because it expires after approximately six days.
