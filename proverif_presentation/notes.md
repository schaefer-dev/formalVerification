Hello everyone, today I will present you guys the ProVerif Verifier

# 2 Motivation
As everyone here probably knows by now, the design of security protocols is a notoriously error-prone task. We already discussed e.g. the famous attack found by Lowe against the Needham-Schroeder public-key protocol, which was found only 17 years after its publication. 
Additionally, it’s an issue that security errors cannot be detected by functional testing, since they appear only in the presence of a malicious adversary. 

It is quite obvious that proving security properties of protocols is very valuable, especially if we don’t have to make very strict assumptions doing so.

In order to verify protocols, there are only two main models that one has to consider:
- In the symbolic model, often called Dolev-Yao model, cryptographic primitives are considered as perfect blackboxes. The symbolic model is an abstract model that makes it easier to build automatic verification tool by arguing about variables instead of fixed values.
- In contrast, in the computational model, messages are described as bitstrings. Cryptographic primitives in the computational Model are functions from bitstrings to bitstrings, and the adversary is any probabilistic Turing machine. This is the model usually considered by cryptographers, because it models the protocol without assuming perfect cryptography. While the computational model is closer to the real execution of protocols it is still a model and additionally the proofs are much harder to automate compared to the Symbolic Model:w


# 3 Proverif
ProVerif takes the approach of the Symbolic Model, just like the previous Approaches we discussed in the preceding talks, but with one very big difference:
The previously presented methods all had one very strict limitation: The number of protocol executions had to be bounded. This was done because if the number of executions of the protocol is not bounded, the problem is undecidable for a reasonable model of protocols. Hence, there exists no automatic tool that always terminates and solves this problem.

ProVerif uses an abstract representation of protocols using so called Horn clauses, which is an improvement on tree-automata because it keeps relational information on messages. However, the problem proverif tries to solve is still undecidable which means that termination is not guaranteed in general, we will get more into that later.


# 4 Overview
This Slide gives a rough outline of how ProVerif works.
One has to input a model of the protocol in an extension of the pi calculus which is similar to the applied pi calculus. 
ProVerif supports a wide variety of cryptographic primitives out of the box, which are modeled either by rewrite rules or by equations. 
ProVerif also takes as input the security properties that we want to prove. It can verify various security properties, including secrecy, authentication, and some observational equivalence properties. 

Everything else is entirely automatic. Proverif translates the protocol description into a set of Horn clauses, the security properties to prove are translated into derivability queries on these clauses, more on that later.

It then tries to derive the fact that would violate the security property we specified earlier. If no such fact is derivable, then the desired security property is proven. If the fact is derivable, then there may be an attack against the considered property. Conveniently the derivation that was used already describes the attack, which has to be checked by hand to make sure that it is indeed valid. 

Possible false attacks are caused by the abstractions that are made when translating the protocol into the set of horn clauses but we will get more into detail on that later in my presentation.


# 5 Input Language Terms
Okay so lets start by looking at the input language that is used to specify Protocols. On this slide you can see a small table that specifies the syntax of terms, which represent data, like for example variables, in our protocol. 

Names on the other hand represent atomic data, such as randomly generated keys and nonces. 

Then we also have constructors, which are used to build terms representing for example more complex data structures such as tuples or cryptographic operations, I have some examples of that in a bit.


# 6 Input Language Processes
This table describes the second part of proverif’s input language called Processes or Programs. Most of the these constructs come from the pi calculus.
This Process here outputs the Message N on the Channel M and then executes P.
This Process here inputs a message on Channel M and binds the variable x to the content of that message and then executes P. The Channel M could be a name or a variable. It is also important to keep in mind that passing of messages is always synchrononous.
The Nil Process written as zero simply does nothing
Parallel Composition obvious
Exclamation Mark means that a process is executed unbounded number of times in parallel, which is e.g. used to represent an unbounded number of protocol executions.
The Restriction Process generates a new name a (remember, this means random value) and executes P. 
The only remaining not obvious Process is the destructor, which is basically a function that manipulates terms in processes. This is easier to understand in some examples combined with constructors.


# 7 Examples Constructor Destructor
I mentioned earlier already that constructors can be used to represent data structors. Here is a constructor called n-touple which creates a touple containing these terms specified in its arguments. The destructor i-th here implements the projection of the i-th element of the touple, so it “returns” the i-th argument of the touple creation.

Similarly one can use constructors to encrypt a message with a secret key y and decrypt it with a destructor taking the same key y to get access to the plaintext m again.

Asymetric encryption is basically the same with the additional constructor ‘pk’ creating a public-key from a given private-key.

The same also goes for signatures on which 2 destructors on the same constructed term make sense, one to return the message only if the signature was valid and one simply to get the message content regardless of the signature being valid or not, useful if you dont have access to the public-key of y in this case.


# 8 Protocol Example Denning-Sacco key distribution
This is an example of how a protocol specification in proverifs input language looks like for a simplified version of the Denning-Sacco-Key distribution protocol, which involves to principals A and B exchanging a secret key s. 
We use a process P0 to generate random values for the secret keys of both A and B and get the corresponding public keys of these keys. The public keys are output on a public channel c and the corresponding secret keys are given only to their respective “owner”.

Process A first receives the public key of B on the public channel before generating a random value k and proceeding with the protocol. Receiving this key is strictly speaking not part of the protocol, but it allows the attacker to control, with whom A is going to execute the session.

This is abused by an attack on this protocol, because the adversary can send its own public key to A, which means that the adversaries public key would be used by A to encrypt the first message. This means that the adversary can also decrypt the message and thus access k. The adversary can now proceed to simply act like B would and send an attacker-controlled value s to A encrypted with k, which A assumes only its valid partner has knowledge of.


# 9 The Adversary
We assume that the protocol we defined is executed in the presence of an adversary that can listen to all messages, compute, and send all messages it has, following the Dolev-Yao model. During its computations, the attacker can apply all constructors and destructors that were previously defined. 

The adversary can be represented by any process that has a set of public names S in its initial knowledge

The Initial knowledge of the adversary contains only names in S, but one can give any terms to the adversary by sending them on a channel in S, similarly to how it was done in the example we just looked at by sending the public keys of A and B on a public Channel c.

Intuitively, we say that a process P preserves the secrecy of M when M cannot be output on a public channel, in a run of P with any adversary. This was violated previously because the adversary would be able to output the supposedly secret values k and s in a run with the protocol like I described earlier


# 10 Horn Clauses
Okay, so know we know how to specify a protocol, such that ProVerif does its job. I already mentioned that internally, ProVerif translates the protocol we specified into a representation of attacker combined with protocol, specified by a set of Horn clauses. Horn clauses are essentially logical formula of a rule-like form. The Syntax of these clauses is described in this table. Important to note is that Names still represent random numbers with the additional guarantee that names created in different runs of the protocol are always distinct. This is modeled by names being functions that take as input the previously exchanged messages with the principal that creates the random value.

A fact F expresses a property of the messages P1,…Pn. The property that was used most often in the paper is the predicate called attacker(p), which describes that attacker may have access to this pattern p.

Horn Clauses themselves simply describe an implication if all Facts F1,… Fn, called hypothesis, are true, then the conclusion F is also true.


# 11 Horn Clauses of Attacker
So we can take a look at how these clauses look like on an example of simple public key cryptography. The horn clauses that would be built from the constructors aenc and pk are shown here. They basically state if the attacker has knowledge of m and pk, then it also has knowledge of this particular encrypted message. Intuitively the adversary applies to constructor aenc to encrypt the message itself. Similarly the adversary learns a valid public-key corresponding to this secret key by applying the constructor pk.

The same applies for the clause corresponding the destructor adec.

The clauses above describe the computation abilities of the attacker. To describe that the attacker initially has knowledge of the public keys of the protocol participants. We add the clauses Horn like this with an empty Hypothesis and only a conclusion.


# 12 Horn Clauses for the Protocol
Remember the example previously where the Attacker was able to decide with which principal A executes the protocol with, here is a quick summary of it again. 
The Clause proverif build here basically means that under the assumption that the adversary has knowledge of a public key pk(x), it also gains knowledge of an encrypted message with a structure like this. 

While this might look unintuitive at first its very easy to understand in such a simple protocol. This Clause exists because we allowed the adversary to send A the public key with which the first message is being encrypted. The clause basically states that if the attacker has knowledge of any public key, it could send this public key to A and A would start the program and send this exact message on a public channel, which means that the adversary gains knowledge of this exact message.


So to summarise this, the protocol can be represented by three sets of Horn clauses:
one defining  the computational abilities of attacker [go back show enc example]
Facts corresponding to the initial knowledge of the attacker [go back show pubk example]
Clauses Representing the messages of the protocol itself [on this slide above]


# 14 Secrecy Criterion
With all the clauses we built the secrecy criterion is quite obvious. We have to check if there is a way for an attacker to gain access to some secret value s, which means that we have to check if the fact ‘attacker(s)’ can be derived. 

If this fact can be derived, the sequence of clauses applied to derive it directly lead to the description of the attack.

So in the next slides we will learn how proverif is able to check if a fact can be derived from the set of clauses that resulted from the translation of our protocol description.
