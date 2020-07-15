---
layout: post
title: Creating a simple Blockchain
description: A blockchain, originally block chain, is a continuously growing list of records, called blocks, which are linked and secured using cryptography.
categories: blockchain
tags: java blockchain
---

<details markdown="1" class="table-of-contents" open>
<summary>Table of contents</summary>
* Table of Contents
{:toc}
</details>

The Wikipedia definition of blockchain says

> A blockchain, originally block chain, is a continuously growing list of records, called blocks, which are linked and secured using cryptography.

Okay, let’s break it down to get a better understanding.

### Blocks

A block contains transaction data, the cryptographic hash of the previous block, the timestamp and a nonce. You must be wondering what a nonce is, It will be explained in detail later, but for now just think of it as a random number.

{:refdef: class="centered-image"}
![Blocks in a blockchain]({{ 'public/images/blockchain-blocks.png' | relative_url }})
{: refdef}

A block’s hash is generated using the transaction data, timestamp, previous block’s hash and the nonce as input.

For example, let’s say the hashing algorithm used is [SHA-256](https://en.wikipedia.org/wiki/Secure_Hash_Algorithms) (more complex hashing algorithms are used in actual implementations) and assume the following as the block’s data.


<table class="table table-striped table-responsive">
    <tr">
        <th class="text-right-align"> Transaction Data </th>
        <td> Hello, World </td>
    </tr>
    <tr>
        <th class="text-right-align"> Timestamp </th>
        <td> 1527696247729 </td>
    </tr>
    <tr>
        <th class="text-right-align">Previous block hash</th>
        <td>5591ae80713bc6324cae319ddfbdbedd710aabb96a33af80b0732ecd8e42314d</td>
    </tr>
    <tr>
        <th class="text-right-align">Nonce</th>
        <td>383212</td>
    </tr>
</table>


We append the all the values to a string, which looks something like this

{% highlight plaintext %}
"Hello, World15276962477295591ae80713bc6324cae319ddfbdbedd710aabb96a33af80b0732ecd8e42314d383212"
{% endhighlight %}

After applying SHA-256 on the above string we’ll get the value (you can experiment with SHA-256 online [here](https://passwordsgenerator.net/sha256-hash-generator/))

{% highlight plaintext %}
"4527f3eb2c927118f1da9ae1327862a1b52d998108ce608526b4814ca1a5af50"
{% endhighlight %}

This is the value which is going to be used in the next block’s previous block hash property.

Enough with the theory, let’s dive into code

First let’s create a helper class to return the hash of a given string using SHA-256 algorithm

{% highlight java %}

import java.security.MessageDigest; 
import java.security.NoSuchAlgorithmException; 
import java.nio.charset.StandardCharsets; 

public class CryptoHelper { 

  public static String sha256(String text) throws NoSuchAlgorithmException { 
    
    MessageDigest digest = MessageDigest.getInstance("SHA-256"); 
    digest.update(text.getBytes(StandardCharsets.UTF_8)); 
    byte[] hash = digest.digest(); 
    
    StringBuffer sb = new StringBuffer(); 
    for (int i = 0; i < hash.length; i++) 
      sb.append(Integer.toString((hash[i] & 0xff) + 0x100, 16).substring(1)); 
    return sb.toString().toUpperCase(); 
  }
}

{% endhighlight %}

Now, let’s create the `Block` class

{% highlight java %}
public class Block {
  public long nonce; 
  private long timestamp; 
  public String transactionData; 
  public String previousBlockHash; 
  
  public Block(String transactionData) { 
    nonce = 0; 
    timestamp = System.currentTimeMillis(); 
    this.transactionData = transactionData; 
  } 
  
  public String getHash() throws NoSuchAlgorithmException { 
    return CryptoHelper.sha256(transactionData + String.valueOf(timestamp) 
      + String.valueOf(previousBlockHash) 
      + String.valueOf(nonce));
  } 
  
  @Override 
  public String toString() { 
    return String.format("%s\t%d\t%d\t%s", previousBlockHash, timestamp, nonce, transactionData); 
  } 
}
{% endhighlight %}

For brevity, `transactionData` is a simple string and also let's not worry about the `nonce` for now, let it be `0`, we'll be coming back to this later.

Let's create a class to represent a collection of blocks,

{% highlight java %}
import java.security.NoSuchAlgorithmException; 
import java.util.ArrayList; 
import java.util.List; 

public class BlockChain { 
  public List<Block> chain; 
  
  public BlockChain() { 
    chain = new ArrayList<>(); 
  } 
  
  public void addBlock(Block block) throws NoSuchAlgorithmException { 
    block.previousBlockHash = chain.size() == 0 ? CryptoHelper.sha256("0") : chain.get(chain.size()-1).getHash(); 
    chain.add(block); 
  } 
  
  public boolean isValid() throws NoSuchAlgorithmException { 
    for (var i=1; i < chain.size(); ++i) { 
      var previousBlock = chain.get(i-1); 
      var currentBlock = chain.get(i); 
      if (!currentBlock.previousBlockHash.equals(previousBlock.getHash())) { 
        return false; 
      } 
    } 
    return true; 
  }
}
{% endhighlight %}

This class is pretty simple contains a List to store blocks and methods to add a block and validate the chain.

Now that we have the foundation classes, let’s create a Driver class for testing this out

{% highlight java %}
public class Main { 
  public static void main(String[] args) { 
    try { 
      var blockChain = new BlockChain(); 
      
      Block block1 = new Block("Hello"); 
      Block block2 = new Block("World"); 
      Block block3 = new Block("Whatever"); 
      
      blockChain.addBlock(block1); 
      blockChain.addBlock(block2); 
      blockChain.addBlock(block3); 
      
      for (Block block: blockChain.chain) { 
        System.out.println(block); 
      } 
      
      System.out.println("Is valid chain : " + blockChain.isValid()); 
    } 
    catch(Exception e) { 
      System.err.println(e); 
    } 
  } 
}
{% endhighlight %}

On a sample run, we get the following values, the first column is the previous block’s hash value, followed by the timestamp, nonce and the transaction data.

```
5FECEB66FFC86F38D952786C6D696C79C2DBC239DD4E91B46729D73A27FB57E9	1527703318986	0	Hello 
E5A6134694C22742C88639C8E3442BDF19FB7843F829B2351B24CB0CCC9AA917	1527703318986	0	World 
759F215F98C2374696552663280BBC90378741DBD36B067A6DF0FAA50DD38769	1527703318986	0	Whatever 
Is valid chain : true
```

From the output, we can see that the blockchain is valid (i,e) The previous block hashes are matching correctly.

Let’s try modifying the `transactionData` of a block

{% highlight java %}
public static void main(String[] args) { 
  try { 
    var blockChain = new BlockChain(); 
    
    Block block1 = new Block("Hello"); 
    Block block2 = new Block("World"); 
    Block block3 = new Block("Whatever"); 
    
    blockChain.addBlock(block1); 
    blockChain.addBlock(block2); 
    blockChain.addBlock(block3); 
    
    for (Block block: blockChain.chain) { 
      System.out.println(block); 
    } 
    
    System.out.println("Is valid chain : " + blockChain.isValid()); 
    
    // modify second block's data.. 
    blockChain.chain.get(1).transactionData = "World1"; 
    System.out.println("Is valid chain : " + blockChain.isValid()); // will be false 
  } 
  catch(Exception e) { 
    System.err.println(e); 
  }
}
{% endhighlight %}


```
5FECEB66FFC86F38D952786C6D696C79C2DBC239DD4E91B46729D73A27FB57E9	1527703318986	0	Hello 
E5A6134694C22742C88639C8E3442BDF19FB7843F829B2351B24CB0CCC9AA917	1527703318986	0	World 
759F215F98C2374696552663280BBC90378741DBD36B067A6DF0FAA50DD38769	1527703318986	0	Whatever 
Is valid chain : true 
Is valid chain : false
```

Indeed, it’s not valid because when we changed the second block’s `transactionData`, the next block's `previousHash` has to be updated to match the new hash. We can solve this easily by recalculating the hashes and assigning the previousHash accordingly. If changing an element is so simple, what about this statement?

> By design, a blockchain is inherently resistant to modification of the data

That’s where the `nonce` comes into play.

### What's a nonce?

The nonce is a number that should be predicted and acts as a [proof of work](https://en.wikipedia.org/wiki/Proof-of-work_system), making any modification in the blockchain very hard

Blockchain implementations have a difficulty level that depends on the implementation, but to make things simpler, let’s say the produced hash should start with a specified number of zeros

For example, When difficulty is 1, the block’s hash should start with 1 zero, when the difficulty is 2, the block’s hash should start with 2 zeros and so on. The only way in which we can produce a hash that satisfies the difficulty is to try changing the nonce value and hope that some number produces a hash that meets the difficulty criteria, this method of trying every possible value to find a solution is called bruteforcing.

To add that functionality to our implementation, let’s start with adding the `mine` method to the `Block` class

{% highlight java %}
public void mine(int difficulty) throws NoSuchAlgorithmException { 
  String pre = ""; 
  for (; pre.length() < difficulty; pre += "0"); 
  while(!getHash().startsWith(pre)) 
    nonce++;
}
{% endhighlight %}

In the method, we are incrementing the value of `nonce` until the hash starts with the number of zeros specified by the `difficulty`. Then we call this method in the BlockChain class before adding a block to the chain

{% highlight java %}
public class BlockChain {
  
  public List<Block> chain; 
  private int difficulty;
  
  public BlockChain(int difficulty) { 
    chain = new ArrayList<>(); 
    this.difficulty = 
    difficulty; 
  }
}
{% endhighlight %}

modifying `addBlock` to mine block before adding to the blockchain

{% highlight java %}
public void addBlock(Block block) throws NoSuchAlgorithmException { 
  block.previousBlockHash = chain.size() == 0 ? CryptoHelper.sha256("0") : chain.get(chain.size()-1).getHash(); 
  
  // before adding a block to the blockchain 
  // mine block for the magic nonce value 
  block.mine(difficulty); 
  chain.add(block); 
}

{% endhighlight %}

also, the `isValid` method needs to check every block to see if it satisfies the difficulty criteria,

{% highlight java %}

public boolean isValid() throws NoSuchAlgorithmException { 
  var pre = ""; 
  for(;pre.length() < difficulty; pre += "0"); 
  for (var i=1; i < chain.size(); ++i) { 
    var previousBlock = chain.get(i-1); 
    var currentBlock = chain.get(i); 
    if (!currentBlock.getHash().startsWith(pre) || !currentBlock.previousBlockHash.equals(previousBlock.getHash())) { 
      return false; 
    } 
  } 
  return true;
}

{% endhighlight %}

And finally, we pass the `difficulty` from the main method

{% highlight java %}

public static void main(String[] args) { 
  try { 
    var difficulty = 5; 
    var blockChain = new BlockChain(difficulty); 
    
    Block block1 = new Block("Hello"); 
    Block block2 = new Block("World"); 
    Block block3 = new Block("Whatever"); 
    
    blockChain.addBlock(block1); 
    blockChain.addBlock(block2); 
    blockChain.addBlock(block3); 
    
    System.out.println(String.format("%64s\t%13s\t%7s\t%s", "Previous Block's hash", 
        "Timestamp", "Nonce", "Transaction Data")); 
    
    for (Block block: blockChain.chain) { 
      System.out.println(block); 
    } 
    
    System.out.println("Is valid chain : " + blockChain.isValid()); 
    blockChain.chain.get(1).transactionData = "World1"; // Modifying block's value here

    System.out.println("Is valid chain : " + blockChain.isValid()); 
  } 
  catch(Exception e) { 
    System.err.println(e); 
  }
}

{% endhighlight %}

On a sample run, we get the following output

{% highlight plaintext %}
Previous Block's hash                                                   Timestamp     Nonce	Transaction Data 
5FECEB66FFC86F38D952786C6D696C79C2DBC239DD4E91B46729D73A27FB57E9	1527709815253 511316	Hello 
00000BE462BE07F21AC57939DC46310322F8ACF170B1C16CC03E3DCC431A59BE	1527709815253 1362348	World 
00000E3D3CFFAEE5E8360EB8E7FF8AB39F270D39BCD5830A8505974632B98DC4	1527709815253 792204	Whatever 
Is valid chain : true 
Is valid chain : false
{% endhighlight %}

Observe the **Nonce** values, for the first block, it took **5,11,316** calls to `getHash()` to find a nonce value that satisfies the difficulty criteria. We have used a difficulty of `5` which is tiny, whereas in actual implementations the difficulty value is much higher and also a much complex hashing algorithm will be used.

Imagine having a chain of about 10 blocks, if block 3 is modified, then the nonce values for blocks starting from 3 need to be recalculated which is a pretty expensive and time consuming operation in terms of CPU power. The nonce is what makes it difficult to modify the blocks in a blockchain. Try tinkering with the difficulty values and you can observe that as the difficulty increases, the time taken to calculate the nonce increases exponentially.

### Links

- [Source Code](https://github.com/cyberpirate92/SimpleBlockChain)
