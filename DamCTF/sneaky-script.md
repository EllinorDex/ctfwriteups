![2021-11-06_16-24-36](https://user-images.githubusercontent.com/84577967/140614866-795e6e02-759e-4322-85fe-6517bc0879b6.png)

One of challenges from the recent CTF [DamCTF](https://damctf.xyz/) and the only one in the malware category.
---
We were given a zip file, after extracting we got 2 files a pcap file and a bash script

![2021-11-06_16-42-44](https://user-images.githubusercontent.com/84577967/140615375-62bf876d-6ee8-4f83-83eb-051c9c99b517.png)

After checking the bash script I came to this line that shows us that the bash script is downloading a python file from a host and executing it.

![2021-11-06_16-20-48](https://user-images.githubusercontent.com/84577967/140615442-3020d258-13cd-4cbb-988a-f7fa3714f7d5.png)

Great ! if we check the host will find it down, but let's go to the pcap file and take a look on the captured trafics will find the base64 encoded file

![2021-11-06_16-20-19](https://user-images.githubusercontent.com/84577967/140615524-cae32560-bb31-400e-ba73-280f338c8ce8.png)

Let's decode it and check it using `file`

![2021-11-06_16-48-20](https://user-images.githubusercontent.com/84577967/140615571-5f638027-cfd4-4b8a-b8e2-ced6a0ff8392.png)

So using [uncompyle6](https://github.com/rocky/python-uncompyle6/) we can decomplie the file 

![2021-11-06_16-23-12](https://user-images.githubusercontent.com/84577967/140615661-19c0c02c-a1a5-4464-a783-1a0a93cbfbdf.png)

So the python file stole some information: `network, process list, ssh keys ...`  and send it to a new host on endpoint `/upload`

![2021-11-06_16-23-32](https://user-images.githubusercontent.com/84577967/140616016-873d16b8-bd62-4271-85ff-eb0c74c67032.png)

But the sent data is encrypted using XOR and the key `8675309`, to decrypt it let's save it in a file and run the python script

```python
import base64

def decrypt():
	r = ""
	with open('encrypted.txt') as f:
		contents = f.read()
		p = base64.b64decode(contents)
		k = b'8675309'
		for i in range(len(p)):
			d = p[i] ^ k[i % len(k)]
			r += chr(d)
		print(r)
decrypt()
```

The flag:

![2021-11-06_17-06-20](https://user-images.githubusercontent.com/84577967/140616098-fba25eb7-0526-44b4-9f48-11243985a779.png)
