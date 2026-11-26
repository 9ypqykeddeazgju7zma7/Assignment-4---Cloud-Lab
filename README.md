# Web Page Querying PostgreSQL on a Separate AWS VM

*Setup guide for a web VM and a database VM in the same subnet*

## Overview

This guide sets up two Ubuntu Server 24.04 EC2 instances in the same VPC subnet:

- **Web VM**: Apache serves one PHP page that queries the database.
- **DB VM**: PostgreSQL accepts connections only from the web VM over private IPs. For this lab it also has a public IP, for SSH and package installs.

Traffic flow: Browser → Web VM (Apache, port 80) → DB VM (PostgreSQL, port 5432, private IP only).

Replace these placeholders with your own values:

| Item | Example value |
| --- | --- |
| Web VM private IP | 10.0.1.10 |
| DB VM private IP | 10.0.1.20 |
| Database name | appdb |
| Database user | appuser |
| Database password | p4ssw0rd |

You can find each instance’s private IP in the EC2 console under the instance details.

## Step 1: Create the two EC2 instances

In the AWS console, open **EC2 → Instances → Launch instances**. Launch two instances, one at a time, with the same OS, key pair, VPC and subnet.

**Web VM**

1. Name: `web-vm`.
2. Application and OS image: **Ubuntu Server 24.04 LTS**.
3. Instance type: `t3.micro` (enough for this guide).
4. Key pair:

   a. Choose **Create new key pair**, name it `webapp-key`, type RSA, and set the private key file format to `.ppk` (the format PuTTY uses).

   b. Save the downloaded `webapp-key.ppk` file. You will reuse this key for both VMs.
5. Network settings:

   a. Click **Edit**, then choose your VPC and a subnet, and remember which subnet you picked.

   b. Set **Auto-assign public IP** to **Enable**.

   c. For the firewall, choose **Select existing security group** and pick **default**. You create and attach the real security groups in Step 2.
6. Storage: leave the default 8 GiB.
7. Click **Launch instance**.

**DB VM**

1. Name: `db-vm`.
2. Use the same image, instance type and key pair (`webapp-key`) as the web VM.
3. Network settings:

   a. Click **Edit** and choose the **same VPC and the same subnet** as the web VM.

   b. Set **Auto-assign public IP** to **Enable**, so you can connect to the DB VM from your computer in this lab.

   c. Choose the **default** security group again.
4. Storage: leave the default 8 GiB (increase it if you expect much more data).
5. Click **Launch instance**.

**Write down the IP addresses**

When both instances show **Running**, select each one and note these values:

- Web VM: private IPv4 address and public IPv4 address
- DB VM: private IPv4 address and public IPv4 address

Use them in place of the example addresses (10.0.1.10 and 10.0.1.20) in the rest of the guide.

## Step 2: Configure security groups

Use two security groups, one per VM.

**Web security group (web-sg)**

- Inbound: TCP 80 and 443 from 0.0.0.0/0
- Inbound: TCP 22 from your own IP only
- Outbound: allow all (default)

**Database security group (db-sg)**

- Inbound: TCP 5432 with the source set to **web-sg** (pick the security group itself, not an IP range)
- Inbound: TCP 22 from your own IP only
- No other inbound rules, and never a rule open to 0.0.0.0/0

The default network ACL already allows traffic inside the subnet.&#32;

Attach each group to its instance: select the instance, then Actions → Security → Change security groups.

**Note on the DB VM’s public IP**

- Keep in mind that normally we would keep the DB VM off the internet by not assigning it a public IP address. In this lab we will assign it a public IP address though so we can install packages and for easy access for the students.

## Step 3: Connect to both VMs with PuTTY

On your Windows computer, use PuTTY to connect to each VM directly. Have your `webapp-key.ppk` file ready, and keep two PuTTY windows open, one for each VM.

**Web VM**

a. Open **PuTTY**. On the **Session** page, enter `ubuntu@<web VM public IP>` as the Host Name. Leave the port at 22 and the connection type as SSH.

b. In the left menu, go to **Connection → SSH → Auth → Credentials** (in older PuTTY versions, **Connection → SSH → Auth**). Next to **Private key file for authentication**, click **Browse** and select `webapp-key.ppk`.

c. Go back to the **Session** page, type `web-vm` in **Saved Sessions**, and click **Save**.

d. Click **Open**. Click **Accept** if PuTTY shows a host key alert. The prompt changes to something like `ubuntu@ip-10-0-1-10`, which means you are now on the web VM.

**DB VM**

a. Open a second PuTTY window. On the **Session** page, enter `ubuntu@<DB VM public IP>` as the Host Name.

b. Under **Connection → SSH → Auth → Credentials**, select the same `webapp-key.ppk` file.

c. Go back to the **Session** page, type `db-vm` in **Saved Sessions**, and click **Save**.

d. Click **Open** and click **Accept** if PuTTY shows a host key alert. The prompt changes to something like `ubuntu@ip-10-0-1-20`, which means you are now on the DB VM.

**Set the hostnames**

a. In the web VM window, run:

```bash
sudo hostname web-vm
```

b. In the DB VM window, run:

```bash
sudo hostname db-vm
```

Run `hostname` to check the result. Your prompt shows the new name the next time you connect. The name lasts until the VM reboots, and `sudo` may print an unable to resolve host warning, which is harmless.

Type `exit` to leave a VM when you are finished. To connect again later, open PuTTY, select the `web-vm` or `db-vm` saved session, and click **Load** and then **Open**.

If PuTTY says No supported authentication methods available or Access denied, check that the user name is `ubuntu` and that the private key file is `webapp-key.ppk`. If the connection times out, check that the security group of that VM allows port 22 from your current IP address (web-sg for the web VM, db-sg for the DB VM).

## Step 4: Install PostgreSQL on the DB VM

a. Connect to the DB VM (see Step 3) and run:

```bash
sudo apt update
sudo apt install -y postgresql
```

b. Make PostgreSQL listen on the private IP and allow only the web VM to connect (use your own IPs):

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET listen_addresses = 'localhost,10.0.1.20';"
echo 'host    appdb    appuser    10.0.1.10/32    scram-sha-256' | sudo tee -a "$(sudo -u postgres psql -t -A -c 'SHOW hba_file;')"
sudo systemctl restart postgresql
```

c. Check that PostgreSQL is listening on the private IP:

```bash
ss -tln
```

You should see 10.0.1.20:5432 in the output.

## Step 5: Create the database, user and a 20-row products table

Run these on the DB VM (change one of the products to be your full name, so we can search for it later and take a screenshot to show that you completed this assignment. For example, you might change Desk Lamp to Bob Smith before pasting the code into PuTTY):

```bash
sudo -u postgres psql <<'SQL'
CREATE USER appuser WITH PASSWORD 'p4ssw0rd';
CREATE DATABASE appdb OWNER appuser;
SQL

sudo -u postgres psql -d appdb <<'SQL'
CREATE TABLE products (
  product_id INTEGER PRIMARY KEY,
  product_name TEXT NOT NULL,
  price NUMERIC(10,2) NOT NULL
);
INSERT INTO products (product_id, product_name, price) VALUES
  (1001, 'Wireless Mouse', 24.99),
  (1002, 'Mechanical Keyboard', 89.50),
  (1003, 'USB-C Hub', 39.95),
  (1004, '27-inch Monitor', 229.00),
  (1005, 'Laptop Stand', 34.99),
  (1006, 'Webcam 1080p', 49.99),
  (1007, 'Noise-Cancelling Headphones', 149.00),
  (1008, 'Bluetooth Speaker', 59.90),
  (1009, 'External SSD 1TB', 99.99),
  (1010, 'Portable Charger', 29.95),
  (1011, 'Desk Lamp', 27.50),
  (1012, 'Ergonomic Chair', 249.00),
  (1013, 'Standing Desk', 399.00),
  (1014, 'Wi-Fi Router', 79.99),
  (1015, 'Smart Plug', 14.99),
  (1016, 'HDMI Cable 2m', 9.99),
  (1017, 'Surge Protector', 22.49),
  (1018, 'USB Microphone', 74.00),
  (1019, 'Graphics Tablet', 119.00),
  (1020, 'Phone Stand', 12.95);
GRANT SELECT ON products TO appuser;
SQL
```

The web app user only gets SELECT, which is enough for a read-only page. Grant INSERT or UPDATE later only if the page needs them.

## Step 6: Test the connection from the web VM

On the web VM, install the PostgreSQL client and query the database:

```bash
sudo apt update
sudo apt install -y postgresql-client
psql -h 10.0.1.20 -U appuser -d appdb -c 'SELECT * FROM products ORDER BY product_id;'
```

Enter the password when asked. You should see all 20 products. If it hangs or fails, see Troubleshooting below before continuing.

## Step 7: Install Apache and PHP, then create the page

a. In your web VM session (see Step 3), install Apache and PHP, and remove the default Apache page (it would otherwise be shown instead of ours):

```bash
sudo apt update
sudo apt install -y apache2 php libapache2-mod-php php-pgsql
sudo rm /var/www/html/index.html
```

b. Create the page (replace the IP and password with your own):

```bash
sudo tee /var/www/html/index.php > /dev/null <<'EOF'
<?php
$q = trim($_GET['q'] ?? '');
$db = new PDO('pgsql:host=10.0.1.20;port=5432;dbname=appdb;connect_timeout=5', 'appuser', 'p4ssw0rd', [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]);
$stmt = $db->prepare('SELECT product_id, product_name, price FROM products WHERE product_name ILIKE :q ORDER BY product_id');
$stmt->execute([':q' => '%' . $q . '%']);
$rows = $stmt->fetchAll(PDO::FETCH_ASSOC);
?>
<!DOCTYPE html>
<html>
<head>
  <meta charset='utf-8'>
  <meta name='viewport' content='width=device-width, initial-scale=1'>
  <title>Products</title>
  <style>
    body { font-family: Arial, sans-serif; background: #f4f6f8; margin: 0; padding: 2rem; color: #222; }
    .card { max-width: 700px; margin: 0 auto; background: #fff; padding: 1.5rem 2rem; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
    h1 { margin-top: 0; color: #1f4e79; }
    form { margin-bottom: 1rem; }
    input[type=text] { padding: 0.5rem; width: 60%; border: 1px solid #bbb; border-radius: 4px; }
    button { padding: 0.5rem 1rem; background: #1f4e79; color: #fff; border: 0; border-radius: 4px; cursor: pointer; }
    table { width: 100%; border-collapse: collapse; }
    th, td { padding: 0.6rem; text-align: left; border-bottom: 1px solid #ddd; }
    th { background: #1f4e79; color: #fff; }
    tr:nth-child(even) td { background: #f4f6f8; }
    th:last-child, td:last-child { text-align: right; }
  </style>
</head>
<body>
  <div class='card'>
    <h1>Products from PostgreSQL</h1>
    <form method='get'>
      <input type='text' name='q' value='<?= htmlspecialchars($q, ENT_QUOTES) ?>' placeholder='Search products by name'>
      <button type='submit'>Search</button>
    </form>
    <?php if (count($rows) === 0): ?>
    <p>No products found.</p>
    <?php else: ?>
    <table>
      <tr><th>Product ID</th><th>Product name</th><th>Price</th></tr>
      <?php foreach ($rows as $r): ?>
      <tr>
        <td><?= htmlspecialchars($r['product_id']) ?></td>
        <td><?= htmlspecialchars($r['product_name']) ?></td>
        <td>$<?= number_format($r['price'], 2) ?></td>
      </tr>
      <?php endforeach; ?>
    </table>
    <?php endif; ?>
  </div>
</body>
</html>
EOF
```

c. Restart Apache:

```bash
sudo systemctl restart apache2
```

## Step 8: Test from your browser

a. Open `http://<web VM public IP>`. You should see a table of the 20 products from the DB VM.

b. To search, type part of a product name (for example mouse) in the box and click Search. The search ignores upper and lower case. Clear the box and search again to see all products.

Tip: attach an Elastic IP to the web VM so its address does not change when it restarts.

## Optional next steps

- **HTTPS:** put an Application Load Balancer with a free ACM certificate in front of the web VM.
- **Secrets:** keep the DB password in AWS Secrets Manager instead of in the page.
- **Backups:** schedule EBS snapshots or `pg_dump` on the DB VM.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `ssh` to the DB VM times out | db-sg has no port 22 rule for your current IP address | Add or update the rule, then connect to the DB VM’s public IP again |
| `apt update` hangs on the DB VM | The DB VM has no public IP, or its subnet has no route to the internet | Check that the DB VM has a public IPv4 address and that the subnet’s route table has a route to an internet gateway |
| `psql` hangs, then times out | db-sg has no 5432 rule from web-sg, or the wrong IP is used | Fix the db-sg inbound rule and use the DB VM’s private IP |
| Connection refused | PostgreSQL is not listening on the private IP | Run `sudo -u postgres psql -c 'SHOW listen_addresses;'`, restart PostgreSQL, and look for 5432 with `ss -tln` |
| no pg\_hba.conf entry for host | The pg\_hba.conf line is missing or has the wrong IP, database or user | Fix the line, then run `sudo systemctl reload postgresql` |
| password authentication failed | Wrong password or user | Run `ALTER USER appuser WITH PASSWORD 'new-password';` as the postgres user and update the page |
| permission denied for table products | Missing GRANT | Run `GRANT SELECT ON products TO appuser;` in appdb as the postgres user |
| Default Apache page still shows | The default index.html still exists | Run `sudo rm /var/www/html/index.html` |
| PHP code shows as text, or could not find driver | The PHP module or php-pgsql is missing | Install the packages, then run `sudo systemctl restart apache2` |
| Blank page or HTTP 500 | PHP or database error | Check `sudo tail -n 20 /var/log/apache2/error.log` |
