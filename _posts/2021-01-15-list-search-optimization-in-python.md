---
layout: post
title:  "Optimizing List Operations in Python"
date:   2021-01-15 18:19:25 +0800
permalink: "/optimizing-list-operations-python"
description: Comparing performance between various ways of conducting list operations in Python.
tags: python dict list optimization
# permalink: /:categories/:year/:month
---

### Search for a Dict in a List of Dicts

Given a list of dictionaries, there are a few ways we can find a dictionary with a particular field. The first iterates through the entire list and sets an equality conditional, the second uses the `next()` function, the third uses the `filter()` function, and the fourth uses list comprehension. The code below shows how these four methods perform on randomly generated data.

```python
import time
import random
import string
import uuid

letters = string.ascii_letters
max_n = 1000000
num_tests = 100

# Section 1: search for a specific dict within a list of dicts where all dicts have the same structure
start = time.time()
listdict = []
for n in range(0, max_n):
	listdict.append({
			'name': ''.join(random.choice(letters) for i in range(10)),
			'uuid': str(uuid.uuid4())
		})
end = time.time()
print('Data generationn: ' + str(end - start))

# Tests
list_found = {
	'selected': [],
	'test 1': [],
	'test 2': [],
	'test 3': [],
	'test 4': []
}

t1 = 0
t2 = 0
t3 = 0
t4 = 0

for t in range(0, num_tests):
	
	# Generate random e to find
	rand_e = listdict[random.randint(0, max_n - 1)]
	list_found['selected'].append(rand_e)

	# Test 1
	start = time.time()
	for n in range(0, max_n):
		if listdict[n]['uuid'] == rand_e['uuid']:
			list_found['test 1'].append(listdict[n])
	end = time.time()
	t1 += end - start

	# Test 2
	start = time.time()
	list_found['test 2'].append(next(x for x in listdict if x['uuid'] == rand_e['uuid']))
	end = time.time()
	t2 += end - start

	# Test 3
	start = time.time()
	list_found['test 3'].append(list(filter(lambda item: item['uuid'] == rand_e['uuid'], listdict))[0])
	end = time.time()
	t3 += end - start

	# Test 4
	start = time.time()
	list_found['test 4'].append([x for x in listdict if x['uuid'] == rand_e['uuid']][0])
	end = time.time()
	t4 += end - start

print('Test 1: ' + str(t1 / num_tests))
print('Test 2: ' + str(t2 / num_tests))
print('Test 3: ' + str(t3 / num_tests))
print('Test 4: ' + str(t4 / num_tests))
```

The results below (run with `n = 1000000` and 100 rounds of tests) indicate the second method is much quicker than any other method.

```
Data generationn: 29.5691080093
Test 1: 0.272169132233
Test 2: 0.0764631557465
Test 3: 0.236747791767
Test 4: 0.206405239105
```

### Difference Between Lists of Dictionaries

Given two lists of dictionaries, there are a few ways we can calculate the difference between these lists. The first is to iterate through one list and use the `not in` operator. As you can imagine, this is quite slow, as it basically does `mn` operations. List comprehension is not much better, although it definitely looks prettier code-wise. The third method involves transforming each list into sets of tuples, finding the difference between those sets, and returning that difference after it's transformed back into a list. Note that this works better if your lists contain unique elements. The code below shows how these three methods perform on randomly generated data.

```python
import time
import random
import string
import uuid

letters = string.ascii_letters
max_n = 50000
num_test_iters = 100

# Section 1: generate two lists, one is a random subset of the other
start = time.time()
listdict = []
for n in range(0, max_n):
	listdict.append({
			'name': ''.join(random.choice(letters) for i in range(10)),
			'uuid': str(uuid.uuid4())
		})
randlist = random.sample(listdict, random.randint(0, max_n - 1))
end = time.time()
print('Gen data time elapsed: ' + str(end - start))

# Test 1: naive method
diff = []
start = time.time()
for n in listdict:
	if n not in randlist:
		diff.append(n)
end = time.time()
print('Method 1 time elapsed: ' + str(end - start))

# Test 2: list comprehension method
start = time.time()
diff = [x for x in listdict if x not in randlist]
end = time.time()
print('Method 2 time elapsed: ' + str(end - start))

#Test 3: set method
start = time.time()
listdict_set = set(tuple(sorted(d.items())) for d in listdict)
randlist_set = set(tuple(sorted(d.items())) for d in randlist)
diff_set = listdict_set.difference(randlist_set)
transformed_back = map(dict, diff_set)
end = time.time()
print('Method 3 time elapsed: ' + str(end - start))
```

The results below (run with `n = 50000`) indicate the third method is much quicker than the first two.

```
Gen data time elapsed: 1.04885411263
Method 1 time elapsed: 136.472131968
Method 2 time elapsed: 136.646327019
Method 3 time elapsed: 0.19629907608
```