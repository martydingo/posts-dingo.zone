---
title: NSUpdate Snippets
tags: nsupdate snippets
excerpt: "i have these snippets handy whenever I need to craft an update to a zone. "
---

# NSUpdate Snippets
---

i have these snippets handy whenever I need to craft an update to a zone. 

Create a text file and paste the desired examples in, then tweak as desired. Finally, run `nsupdate -k ${tsig-key} ${nsupdate-command-example-textfile}` to send the update through to the nameserver

Make sure your nameserver ACL's allow updates from whereever you're executing nsupdate from. 

## Add Record

To add a record. 

### Dry Run 
```
server ${primary_nameserver_ip}
zone ${zone}
update add ${qname} ${ttl} ${type} ${rdata}
show
send
```

### Real Run
```
server ${primary_nameserver_ip}
zone ${zone}
update add ${qname} ${ttl} ${type} ${rdata}
show
send
answer
```

## Delete Record

To delete a record...

### Dry Run
```
server ${primary_nameserver_ip}
zone ${zone}
update add ${qname} ${ttl} ${type} ${rdata}
show
send
```

### Real Run
```
server ${primary_nameserver_ip}
zone ${zone}
update delete ${qname} ${ttl} ${type} ${rdata}
show
send
answer
```