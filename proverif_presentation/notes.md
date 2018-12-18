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


# 15 Resolution
The problem of  determining whether a given fact can be derived from a set of clauses is exactly the problem solved by usual Prolog systems. However, we cannot use such systems here, because they would not terminate, which is quite obvious looking at this clause which would lead to considering more and more complex terms with an unbounded number of encryptions. 

Basically it would encrypt a message, then encrypt the encrypted message again and so on.

So the main idea of ProVerif’s approach is to instead combine pairs of clauses by resolution, and to guide this resolution process by a clever free selection function.


# 16 Resolution 2
This is the formal definition of resolution given in the paper, you don’t have to look at it now - its much easier to grasp with a simple example. We have two clauses R and R’ here. The Idea of resolution is to combine these two clauses into one, which represents the application of one after the other. To achieve this, the selection function selects a Fact, called F0, in the Hypothesis of the second Clause and as you can see this fact is part of R’s conclusion but in a more specified version.
This means that we can unify them using a function sigma.

# 17 Resolution 3
This function sigma describes the most general unifier of the selected clause and the Conclusion of the first Clause. Which is denoted by the colours here. This means that we map all variables in the selected hypothesis of the second clause to the terms their variables are replaced with in the conclusion of the first Clause R. 


# 18 Resolution 4
Using this unifier function sigma we can built the resulting clause by connecting the Hypothesis of the first clause with the remaining hypothesises of the second clause that were not previously selected. The conclusion remains the conclusion of the second Clause.

It is important to keep in mind, that the unifier function has to be applied to all parts of the resulting clause. In this case that meant that we have to replace the ‘m’ in the conlusion of the second Clause with the signed message sigma maps it to. Same applies to sk being mapped to x.

What happened here is the basic idea of how to combine two clauses with each other using resolution with a free selection function.


# 19 Saturate
Obviously all this happens automatically in ProVerif for all combinations of Clauses with different selected Facts. All this is the so called ‘saturate’ phase. In this phase ProVerif basically tries to minimize the set of clauses until no selection of facts in a clauses hypothesis would allow any other clause to be unified with it.

In more detail this means that we start by removing all subsumed clauses from the set of clauses that our protocol and attacker got translate to. 
This basically means that we eliminate clauses that are already expressed inside other clauses.
Afterwards we add new clauses that can be created using resolution, like we did in the example on the previous slide. After each clause that is being added, we remove all subsumed clauses again. We repeat this process of unifying clauses until we reach a fixpoint.
What remains is a set of clauses in which we can not select any hypothesis to unify them with any other clauses.

This saturate algorithm transforms the initial set of clauses into a new one that still derives the same facts, which means that saturate is both sound and complete.


# 20 Algorithm 
This saturate function is the first essential step in ProVerif’s Algorithm. The resulting set of clauses is passed to the so called derivation search together with a fact that depends on the secrecy criterion we would like to proof.

This input Fact F of the derivation search specifies the fact we try to derive from the set of clauses R1. So in our example it would be attacker(s), to check whether or not the attacker is able to gain knowledge of some secret value s.

# 21 Derivation Search
The derivation search performs backward depth first search on applications of the resulting set of clauses. 
If such a derivation result exists, it also enables proverif to reconstruct how the attack happened using the derivation tree.

If no such derivation exists, we have proven our secrecy criterion holds on our simplified set of clauses, and because of the soundness and completeness of saturate, this criterion also holds on the set of clauses that were directly translated from our protocol specification.


# 22 Termination
I already mentioned that ProVerif does not guarantee termination in general. However in practice it terminates in most examples according to the paper. Blanchet and Podelski were able to prove that proverif always terminates on the so called set of ‘tagged Protocols’.

They consider protocols that use as cryptographic primitives only public-key encryption and signatures with atomic keys, shared-key encryption, message authentication codes, and hash functions. Basically, a protocol is tagged when each application of a cryptographic primitive is marked with a distinct constant tag. It is easy to transform a protocol into a tagged protocol by adding tags and generally considered good practice as it avoids type-flaw attacks that could be enabled by confusion between messages.

Most importantly the addition of tagging to a protocol preserves the expected behaviour of the protocol.


# 23 Limitations of Proverif
Like I mentioned multiple times already the main limitation of ProVerif is that termination is not guaranteed unless working on a tagged protocol, as shown on the previous slide.
The Possibility of false attacks being considered by ProVerif is also tedious, because it forces a human to look at the attack output to decide whether it is valid or not. False attacks appear in particular for protocols with temporary secrets: when some value first needs to be kept secret and is revealed later in the protocol, the Horn clause model considers that this value can be reused in the beginning of the protocol, thus breaking the protocol

The main reason for such false attacks are the details that are being lost when translating the specified protocol into a set of horn clauses.
Specifically, the number of repetitions of each action is ignored, since Horn clauses can be applied any number of times. So a step of the protocol can be completed several times, as long as the previous steps have been completed at least once between the same principals.


# 24 Advantages of Proverif
However, the important point is that the approximations are sound: if an attack exists in a more precise model, such as the applied pi calculus or multiset rewriting, then it also exists in the Horn clause representation. 

Performing approximations enables ProVerif to build a much more efficient verifier, which will be able to handle larger and more complex protocols. Another advantage is that the verifier does not have to limit the number of runs of the protocol. Which makes it perfectly suited for the Certification of protocols, because a proof performed by proverif holds regardless of the number of protocol-runs and message size.

The approximations we performed are the key to achieve such strong guarantees and circumvent the state space explosion problem most verifiers struggle with by abstracting much further away from the protocol itself.

Proverif is also considered a very flexible verifier because it is able to prove various security properties.

Can handle a wide range of cryptographic primitives, specified by rewrite rules or by equations. 

ProVerif was also extended to allow for input of Equational theories, which is necessary to model special properties of some cryptography, e.g. modular exponentiation used in a Diffie-Hellman key exchange. This is a big deal because without equational theories we would not be able to specify protocols which use modular exponentiation or e.g. the application of a XOR operation, because both can not be directly represented by a constructor or destructor definition. These Equational theories are automatically translated into rewrite rules by proverif so they require no additional work for the user.



# 26 Optimisations
Because I still have some time left for my presentation I would like to take a look at some optimisations proverif performs during the saturate process we discussed earlier to increase both the performance of the algorithm and the likelihood to terminate. The optimisations are part of the reason ProVerif is considered a very efficient verifier.

The most obvious optimisation is that all tautologies are removed. A tautology is a clause that trivially holds because its conclusion is already part of the hypothesis, so obviously we don’t need to keep those clauses.

Another obvious one is the removal of duplicate hypothesis, to make sure that each  part of the hypothesis is unique and not repeated unnecessarily.

Both these optimisations can be considered standart for logic operations.


# 27 Optimisations 2
However there are also more sophisticated optimisations performed by proverif which are specific to protocols.

The first of those optimisation is the removal of hypothesis attacker(x) when x does not appear anywhere else in the clause. We are able to do this, because this hypothesis attacker(x) would intuitively mean that the attacker has access to at least one message of some value x. It is clear that this assumption trivially holds and does not make much sense without x appearing somewhere else in the clause.

The Elimination of all clauses containing hypothesis attacker(s) is also very important as it removes a lot of clauses entirely for many different protocols. Intuitively the removal of all clauses containing this hypothesis means that we do not consider any clauses that already assume that the attacker has access to some secret information, as that would mean that the protocol is already broken. It is important to note, that this optimisation never causes proverif to wrongly claim that a protocol is secure.

The last optimization that is performed is the decomposition of data constructors. For any data constructor, that is contained in the protocol specification (for example touples), proverif translates it into these two Horn clauses.
The first clause simply defines the computational ability of the attacker to apply this constructor and the second clause defines that attacker(f(p1, . . . , pn)) is derivable if and only if ∀i ∈ {1, . . . , n}, attacker(xi) is derivable.


# 28 Optimisations 3
Now what proverif does, is it replaces all facts of this form, with a fact of this form. And if this replacement happens in a conclusion of a clause, like it does for our first example clause, we generate N clauses instead for each separate attacker(Xi).

This would result in this set of Clauses for first example clause.
And this Clause for our second clause.

This replacement is done recursively, so if there is another constructor in Xi, this one is replaced again and so on.

As you can already notice in this example, this optimisations causes many clauses to be eliminated, because they become tautologies or are subsumed by other clauses.



# 30 Demo
If you are still interested in the ProVerif Verifier I can highly recommend taking a look at this website.
It allows you to load many already defined security protocols (secure ones and flawed ones) and you can change details of those protocols or simply enter any protocol of your choice manually and try to verify it using ProVerif.

I would highly recommend looking at the HTML output of the demo instead of the commandline output for easier readability.
