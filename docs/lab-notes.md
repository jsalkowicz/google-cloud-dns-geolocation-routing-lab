# Lab Notes — Cloud DNS Geolocation Routing

## Goal

Build a small multi-region environment and test how Cloud DNS GEO routing changes DNS answers based on the location of the client.

## Environment

### Clients

- `us-client-vm` — `us-east4-b`
- `europe-client-vm` — `europe-central2-b`
- `asia-client-vm` — `asia-east1-b`

### Web Servers

- `us-web-vm` — `us-east4-b`
- `europe-web-vm` — `europe-central2-b`

Both web servers ran Apache and returned a page that identified the region serving the request.

## APIs

I enabled:

```bash
gcloud services enable compute.googleapis.com
gcloud services enable dns.googleapis.com
```

I verified the services were enabled before moving on.

## Firewall

I created an IAP-based SSH/ICMP rule for the client connections and an HTTP rule for the web servers.

The HTTP rule targeted VMs with the `http-server` network tag.

## Web Server Addresses

I saved the internal addresses as environment variables:

```bash
export US_WEB_IP=$(gcloud compute instances describe us-web-vm   --zone=us-east4-b   --format="value(networkInterfaces.networkIP)")

export EUROPE_WEB_IP=$(gcloud compute instances describe europe-web-vm   --zone=europe-central2-b   --format="value(networkInterfaces.networkIP)")
```

## Private DNS Zone

I created a private `example.com` zone attached to the default VPC.

```bash
gcloud dns managed-zones create example   --description=test   --dns-name=example.com   --networks=default   --visibility=private
```

## GEO Record

I created `geo.example.com` as an A record with a 5-second TTL and GEO routing.

The two routing targets were:

- US region → US web server
- Europe region → Europe web server

## Testing

I connected to each client VM through IAP and repeatedly requested the same DNS name:

```bash
for i in {1..10}; do
  echo $i
  curl geo.example.com
  sleep 6
done
```

The 6-second delay was longer than the 5-second TTL so each test could get a fresh DNS answer instead of relying on a cached response.

### Europe

All ten responses came from the Europe web server.

### United States

All ten responses came from the US web server.

### Asia

No Asia server existed in the GEO policy. Cloud DNS used its nearest-match behavior.

In this lab run, all ten Asia requests were served by the US web server.

## Takeaway

The same hostname can lead clients to different server addresses depending on where the DNS request originates.

The Asia test was the most useful part for me because it showed that GEO routing can still return a result when there is no exact regional entry in the policy.
