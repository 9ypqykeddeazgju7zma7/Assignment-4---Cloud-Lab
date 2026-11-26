# Web Page Querying PostgreSQL on a Separate AWS VM

*Setup guide for a web VM and a database VM in the same subnet*

## Overview

This guide sets up two Ubuntu Server 24.04 EC2 instances in the same VPC subnet:

- **Web VM**: Apache serves one PHP page that queries the database.
- **DB VM**: PostgreSQL runs without a public IP and accepts connections only from the web VM over private IPs.

Traffic flow: Browser → Web VM (Apache, port 80) → DB VM (PostgreSQL, port 5432, private IP only).

Replace these placeholders with your own values:

| Item | Example value |
| --- | --- |
| Web VM private IP | 10.0.1.10 |
| DB VM private IP | 10.0.1.20 |
| Database name | appdb |
| Database user | appuser |
| Database password | ChangeMe-Strong-Password |

You can find each instance’s private IP in the EC2 console under the instance details.

## Step 1: Configure security groups

Use two security groups, one per VM.

**Web security group (web-sg)**

- Inbound: TCP 80 and 443 from 0.0.0.0/0
- Inbound: TCP 22 from your own IP only
- Outbound: allow all (default)

**Database security group (db-sg)**

- Inbound: TCP 5432 with the source set to **web-sg** (pick the security group itself, not an IP range)
- Inbound: TCP 22 with the source set to **web-sg** (you reach the DB VM by jumping through the web VM, see below)
- No other inbound rules, and never a rule open to 0.0.0.0/0

The default network ACL already allows traffic inside the subnet. If you customized your NACLs, allow TCP 5432 and the ephemeral ports (1024-65535) between the two private IPs.

Attach each group to its instance: select the instance, then Actions → Security → Change security groups.

**Keep the DB VM off the internet**

- Launch the DB VM with Auto-assign public IP set to Disable, so it only has a private IP.
- Connect to it by jumping through the web VM: `ssh -J ubuntu@<web VM public IP> ubuntu@10.0.1.20` (add `-i your-key.pem` if needed).
- Installing packages needs outbound internet, so associate an Elastic IP with the DB VM only while you run Steps 2 and 3, then disassociate and release it. The security group still blocks all inbound internet traffic during that time.
- When you are done, the EC2 console should show no public IPv4 address on the DB VM.

## Step 2: Install PostgreSQL on the DB VM

Connect to the DB VM (see Step 1) and run:

```bash
sudo apt update
sudo apt install -y postgresql
```

Next, make PostgreSQL listen on the private IP and allow only the web VM to connect (use your own IPs):

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET listen_addresses = 'localhost,10.0.1.20';"
echo 'host    appdb    appuser    10.0.1.10/32    scram-sha-256' | sudo tee -a "$(sudo -u postgres psql -t -A -c 'SHOW hba_file;')"
sudo systemctl restart postgresql
```

Check that PostgreSQL is listening on the private IP:

```bash
ss -tln
```

You should see 10.0.1.20:5432 in the output.

## Step 3: Create the database, user and a 20-row products table

Run these on the DB VM:

```bash
sudo -u postgres psql <<'SQL'
CREATE USER appuser WITH PASSWORD 'ChangeMe-Strong-Password';
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

## Step 4: Test the connection from the web VM

On the web VM, install the PostgreSQL client and query the database:

```bash
sudo apt update
sudo apt install -y postgresql-client
psql -h 10.0.1.20 -U appuser -d appdb -c 'SELECT * FROM products ORDER BY product_id;'
```

Enter the password when asked. You should see all 20 products. If it hangs or fails, see Troubleshooting below before continuing.

## Step 5: Install Apache and PHP, then create the page

On the web VM, install Apache and PHP, and remove the default Apache page (it would otherwise be shown instead of ours):

```bash
sudo apt update
sudo apt install -y apache2 php libapache2-mod-php php-pgsql
sudo rm /var/www/html/index.html
```

Create the page (replace the IP and password with your own):

```bash
sudo tee /var/www/html/index.php > /dev/null <<'EOF'
<?php
$q = trim($_GET['q'] ?? '');
$db = new PDO('pgsql:host=10.0.1.20;port=5432;dbname=appdb;connect_timeout=5', 'appuser', 'ChangeMe-Strong-Password', [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]);
$stmt = $db->prepare('SELECT product_id, product_name, price FROM products WHERE product_name ILIKE :q ORDER BY product_id');
$stmt->execute([':q' => '%' . $q . '%']);
$rows = $stmt->fetchAll(PDO::FETCH_ASSOC);
?>
<!DOCTYPE html>
<html>
<head>
  <meta charset='utf-8'>
  <title>Products</title>
</head>
<body>
  <h1>Products from PostgreSQL</h1>
  <form method='get'>
    <input type='text' name='q' value='<?= htmlspecialchars($q, ENT_QUOTES) ?>' placeholder='Search products by name'>
    <button type='submit'>Search</button>
  </form>
  <?php if (count($rows) === 0): ?>
  <p>No products found.</p>
  <?php else: ?>
  <table border='1' cellpadding='6'>
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
</body>
</html>
EOF
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

## Step 6: Test from your browser

Open `http://<web VM public IP>`. You should see a table of the 20 products from the DB VM.

To search, type part of a product name (for example mouse) in the box and click Search. The search ignores upper and lower case. Clear the box and search again to see all products.

Tip: attach an Elastic IP to the web VM so its address does not change when it restarts.

## Optional next steps

- **HTTPS:** put an Application Load Balancer with a free ACM certificate in front of the web VM.
- **Secrets:** keep the DB password in AWS Secrets Manager instead of in the page.
- **Backups:** schedule EBS snapshots or `pg_dump` on the DB VM.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `ssh` to the DB VM times out | db-sg has no port 22 rule from web-sg, or you connected directly instead of through the web VM | Add the rule and connect with `ssh -J` through the web VM |
| `apt update` hangs on the DB VM | The DB VM has no outbound internet | Associate a temporary Elastic IP, then release it after Step 3 |
| `psql` hangs, then times out | db-sg has no 5432 rule from web-sg, or the wrong IP is used | Fix the db-sg inbound rule and use the DB VM’s private IP |
| Connection refused | PostgreSQL is not listening on the private IP | Run `sudo -u postgres psql -c 'SHOW listen_addresses;'`, restart PostgreSQL, and look for 5432 with `ss -tln` |
| no pg\_hba.conf entry for host | The pg\_hba.conf line is missing or has the wrong IP, database or user | Fix the line, then run `sudo systemctl reload postgresql` |
| password authentication failed | Wrong password or user | Run `ALTER USER appuser WITH PASSWORD 'new-password';` as the postgres user and update the page |
| permission denied for table products | Missing GRANT | Run `GRANT SELECT ON products TO appuser;` in appdb as the postgres user |
| Default Apache page still shows | The default index.html still exists | Run `sudo rm /var/www/html/index.html` |
| PHP code shows as text, or could not find driver | The PHP module or php-pgsql is missing | Install the packages, then run `sudo systemctl restart apache2` |
| Blank page or HTTP 500 | PHP or database error | Check `sudo tail -n 20 /var/log/apache2/error.log` |
