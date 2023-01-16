# Knows More

## Main features

* [x] Import NTLM Hashes from impacket secretsdump.py output txt file
* [x] Import NTLM Hashes from NTDS.dit and SYSTEM
* [x] Import Cracked NTLM hashes from hashcat output file
* [x] Import BloodHound ZIP or JSON file
* [X] BloodHound importer (import JSON to Neo4J without BloodHound UI)
* [x] Analyse the quality of password (length , lower case, upper case, digit, special and latin)
* [x] Analyse similarity of password with company and user name
* [x] Search for users, passwords and hashes
* [x] Export all cracked credentials direct to BloodHound Neo4j Database as 'owned object'
* [x] Other amazing features...

## Getting stats

```bash
knowsmore --stats
```

This command will produce several statistics about the passwords like the output bellow

```bash
KnowsMore v0.1.4 by Helvio Junior
Active Directory, BloodHound, NTDS hashes and Password Cracks correlation tool
https://github.com/helviojunior/knowsmore
    
 [+] Startup parameters
     command line: knowsmore --stats 
     module: stats
     database file: knowsmore.db
  
 [+] start time 2023-01-11 03:59:20
[?] General Statistics
+-------+----------------+-------+
|   top | description    |   qty |
|-------+----------------+-------|
|     1 | Total Users    | 95369 |
|     2 | Unique Hashes  | 74299 |
|     3 | Cracked Hashes | 23177 |
|     4 | Cracked Users  | 35078 |
+-------+----------------+-------+

 [?] General Top 10 passwords
+-------+-------------+-------+
|   top | password    |   qty |
|-------+-------------+-------|
|     1 | password    |  1111 |
|     2 | 123456      |   824 |
|     3 | 123456789   |   815 |
|     4 | guest       |   553 |
|     5 | qwerty      |   329 |
|     6 | 12345678    |   277 |
|     7 | 111111      |   268 |
|     8 | 12345       |   202 |
|     9 | secret      |   170 |
|    10 | sec4us      |   165 |
+-------+-------------+-------+

 [?] Top 10 weak passwords by company name similarity
+-------+--------------+---------+----------------------+-------+
|   top | password     |   score |   company_similarity |   qty |
|-------+--------------+---------+----------------------+-------|
|     1 | company123   |    7024 |                   80 |  1111 |
|     2 | Company123   |    5209 |                   80 |   824 |
|     3 | company      |    3674 |                  100 |   553 |
|     4 | Company@10   |    2080 |                   80 |   329 |
|     5 | company10    |    1722 |                   86 |   268 |
|     6 | Company@2022 |    1242 |                   71 |   202 |
|     7 | Company@2024 |    1015 |                   71 |   165 |
|     8 | Company2022  |     978 |                   75 |   157 |
|     9 | Company10    |     745 |                   86 |   116 |
|    10 | Company21    |     707 |                   86 |   110 |
+-------+--------------+---------+----------------------+-------+

```

## Installation

```bash
pip3 install --upgrade knowsmore
```

# Execution Flow

There is no an obligation order to import data, but to get better correlation data we suggest the following execution flow:

1. Create database file
2. Import BloodHound files
   1. Domains
   2. GPOs
   3. OUs
   4. Groups
   5. Computers
   6. Users
3. Import NTDS file
4. Import cracked hashes

## Create database file

All data are stored in a SQLite Database

```bash
knowsmore --create-db
```


## Importing BloodHound files

We can import all full BloodHound files into KnowsMore, correlate data, and sync it to Neo4J BloodHound Database. So you can use only KnowsMore to import JSON files directly into Neo4j database instead of use `extremely slow BloodHound User Interface`

```bash
# Bloodhound ZIP File
knowsmore --bloodhound --import-data ~/Desktop/client.zip

# Bloodhound JSON File
knowsmore --bloodhound --import-data ~/Desktop/20220912105336_users.json
```

**Note:** The KnowsMore is capable to import BloodHound ZIP File and JSON files, but we recommend to use ZIP file

### Sync data to Neo4j BloodHound database

```bash
# Bloodhound ZIP File
knowsmore --bloodhound --sync 10.10.10.10:7687 -d neo4j -u neo4j -p 12345678
```

**Note:** The KnowsMore implementation od bloodhount-importer was inpired from [Fox-It BloodHound Import](https://github.com/fox-it/bloodhound-import) implementation. We implemented several changes to save all data in KnowsMore SQLite database and after that do an incremental sync to Neo4J database. With this strategy we have several benefits such as at least 10x faster tham original BloodHound User interface.

## Importing NTDS file

### Option 1

**Note:** Import hashes and clear-text passwords directly from NTDS.dit and SYSTEM registry

```bash
knowsmore --secrets-dump -target LOCAL -ntds ~/Desktop/ntds.dit -system ~/Desktop/SYSTEM
```

### Option 2

**Note:** First use the secretsdump to extract ntds hashes with the command bellow

```bash
secretsdump.py -ntds ntds.dit -system system.reg -hashes lmhash:ntlmhash LOCAL -outputfile ~/Desktop/client_name
```

After that import

```bash
knowsmore --ntlm-hash --import-ntds ~/Desktop/client_name.ntds
```

## Importing cracked hashes

### Cracking hashes

In order to crack the hashes i usualy use hashcat with the command bellow

```bash
# Extract NTLM hashes from file
cat ~/Desktop/client_name.ntds | cut -d ':' -f4 > ntlm_hashes.txt

# Wordlist attack
hashcat -m 1000 -a 0 -O -o "~/Desktop/cracked.txt" --remove "~/Desktop/ntlm_hash.txt" "~/Desktop/Wordlist/*"

# Mask attack
hashcat -m 1000 -a 3 -O --increment --increment-min 4 -o "~/Desktop/cracked.txt" --remove "~/Desktop/ntlm_hash.txt" ?a?a?a?a?a?a?a?a
```

### importing hashcat output file

```bash
knowsmore --ntlm-hash --company clientCompanyName --import-cracked ~/Desktop/cracked.txt
```

**Note:** Change **clientCompanyName** to name of your company

## Wipe sensitive data

As the passwords and his hashes are extremely sensitive data, there is a module to replace the clear text passwords and respective hashes.

**Note:** This command will keep all generated statistics and imported user data.

```bash
knowsmore --wipe
```

## BloodHound Mark as owned

Integrate all credentials cracked to Neo4j Bloodhound database

```bash
knowsmore --bloodhound --mark-owned 10.10.10.10 -d neo4j -u neo4j -p 123456
```

To remote connection make sure that Neo4j database server is accepting remote connection.
Change the line bellow at the config file **/etc/neo4j/neo4j.conf** and restart the service.

```
server.bolt.listen_address=0.0.0.0:7687
```

# To do

[Check the TODO file](TODO.md)