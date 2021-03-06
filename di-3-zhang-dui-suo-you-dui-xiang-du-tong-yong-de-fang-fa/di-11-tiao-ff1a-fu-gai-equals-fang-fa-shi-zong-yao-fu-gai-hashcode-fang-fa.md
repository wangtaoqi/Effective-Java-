### 第11条：覆盖equals方法时总要覆盖hashCode方法

**You must override hashCode in every class that overrides equals.** If you fail to do so, your class will violate the general contract for hashCode, which will prevent it from functioning properly in collections such as HashMap and HashSet. Here is the contract, adapted from the Object specification :

* When the hashCode method is invoked on an object repeatedly during an execution of an application, it must consistently return the same value, provided no information used in equals comparisons is modified. This value need not remain consistent from one execution of an application to another.

* If two objects are equal according to the equals\(Object\) method, then calling hashCode on the two objects must produce the same integer result.

* If two objects are unequal according to the equals\(Object\) method, it is not required that calling hashCode on each of the objects must produce distinct results. However, the programmer should be aware that producing distinct results for unequal objects may improve the performance of hash tables.

**在每个覆盖了equals方法的类里，我们也必须覆盖hashCode方法。**如果我们不这么做，我们的类将会违反hashCode的通用约定，而这也将导致我们的类无法在诸如HashMap和HashSet这样的集合中正常运作。下面是约定的内容，依据于Object规范：

* 只要equals方法里用到的信息没有被更改，那么在一个应用的执行期间，不管调用hashCode方法多少次，hashCode方法都必须持续返回相同的值。但在一个应用程序的多次执行过程中，这个值可以不一致。
* 若两个对象根据equals\(Object\)方法比较是相等的，那么分别调用这两个对象的hashCode方法时必须产生相同的整型数值。
* 若两个对象根据equals\(Object\)方法比较是不等的，那么分别调用这两个对象的hashCode方法时不一定要产生不同的数值。只是程序员应该知道，对于不等的对象若能产生不同的哈希值，有助于提高哈希表的性能。

The key provision that is violated when you fail to override hashCode is the second one: equal objects must have equal hash codes. Two distinct instances may be logically equal according to a class’s equals method, but to Object’s hashCode method, they’re just two objects with nothing much in common. Therefore, Object’s hashCode method returns two seemingly random numbers instead of two equal numbers as required by the contract.

因没有覆盖hashCode方法而违反约定的关键条约是第二条：相等的对象必须有着相同的哈希码（hash code）。根据类的equals方法，两个截然不同的实例在逻辑上可以是相等的，但对于Object的hashCode方法来说，它们也只是两个没有共同之处的对象。因此，Object的hashCode方法会根据两个对象分别返回两个看起来随机的数值，而不是像约定要求的那样返回两个相等的数值。

For example, suppose you attempt to use instances of the PhoneNumber class from Item 10 as keys in a HashMap:

例如，假设我们想将条目10里的PhoneNumber类的实例作为一个HashMap的键：

```
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "Jenny");
```

At this point, you might expect m.get\(new PhoneNumber\(707, 867, 5309\)\) to return "Jenny", but instead, it returns null. Notice that two PhoneNumber instances are involved: one is used for insertion into the HashMap, and a second, equal instance is used for \(attempted\) retrieval. The PhoneNumber class’s failure to override hashCode causes the two equal instances to have unequal hash codes, in violation of the hashCode contract. Therefore, the get method is likely to look for the phone number in a different hash bucket from the one in which it was stored by the put method. Even if the two instances happen to hash to the same bucket, the get method will almost certainly return null, because HashMap has an optimization that caches the hash code associated with each entry and doesn’t bother checking for object equality if the hash codes don’t match.

在这个时候，你可能会期望m.get\(new PhoneNumber\(707, 867, 5309\)\)会返回“Jenny”，然而，它返回了null。我们注意到，这里涉及到两个PhoneNumber实例：一个用来插入HashMap，另一个相等的实例用来（试图用来）获取值。PhoneNumber没有覆盖hashCode导致两个相等的实例没有相等的哈希码，这违反了hashCode约定。因此，put方法将电话号码放入了一个哈希桶（hash bucket）里，而get方法却去一个不同的哈希桶里寻找。即使这两个对象恰好都哈希进了同一个桶，get方法还是很有可能会返回null，因为HashMap有一项优化，它会将每个项关联的哈希码缓存起来，假如传进来的哈希码不匹配，则不校验对象是否相等。

Fixing this problem is as simple as writing a proper hashCode method for PhoneNumber. So what should a hashCode method look like? It’s trivial to write a bad one. This one, for example, is always legal but should never be used:

修复这个问题很简单，只需为PhoneNumber写一个恰当的hashCode方法即可。所以应该如何编写这个hashCode方法？写一个不好的那还不容易。下面这个就是一个合法但永远不会被使用的例子：

```
@Override 
public int hashCode() { 
    return 42; 
}
```

It’s legal because it ensures that equal objects have the same hash code. It’s atrocious because it ensures that every object has the same hash code. Therefore, every object hashes to the same bucket, and hash tables degenerate to linked lists. Programs that should run in linear time instead run in quadratic time. For large hash tables, this is the difference between working and not working.

上面的例子是合法的，因为它保证了相同对象拥有相同的哈希码。但它也保证了每个对象都拥有相同的哈希码，这就糟糕了。因此，每个对象都哈希进了相同的桶，哈希表也退化成链表了。本来程序应该以线性级时间运行的，这下变成了以平方级时间运行。对于大规模的哈希表，这关乎到哈希表能否正常工作。

A good hash function tends to produce unequal hash codes for unequal instances. This is exactly what is meant by the third part of the hashCode contract. Ideally, a hash function should distribute any reasonable collection of unequal instances uniformly across all int values. Achieving this ideal can be difficult. Luckily it’s not too hard to achieve a fair approximation. Here is a simple recipe:

1. Declare an int variable named result, and initialize it to the hash code c for the first significant field in your object, as computed in step 2.a. \(Recall from Item 10 that a significant field is a field that affects equals comparisons.\)
2. For every remaining significant field f in your object, do the following:
    a. Compute an int hash code c for the field:  
    i. If the field is of a primitive type, compute Type.hashCode\(f\), where Type is the boxed primitive class corresponding to f’s type.  
    ii. If the field is an object reference and this class’s equals method compares the field by recursively invoking equals, recursively invoke hashCode on the field. If a more complex comparison is required, compute a “canonical representation” for this field and invoke hashCode on the canonical representation. If the value of the field is null, use 0 \(or some other constant, but 0 is traditional\).  
    iii. If the field is an array, treat it as if each significant element were a separate field. That is, compute a hash code for each significant element by applying these rules recursively, and combine the values per step 2.b. If the array has no significant elements, use a constant, preferably not 0. If all elements are significant, use Arrays.hashCode.  
    b. Combine the hash code c computed in step 2.a into result as follows: result = 31 \* result + c
3. Return result.

一个好的哈希函数会为不相等的实例生成不相等的哈希码。这也正是hashCode约定的第三部分的内容。理想的情况下，哈希函数应该在所有的int值上均匀分布任意不等实例的集合。要达到这种理想状态比较困难。但幸运的是，若我们想近似地达到这种理想状态倒不难。下面给出一种简单的办法：

1. 声明一个叫result的int类型变量，然后将它初始化成对象里第一个重要属性的哈希码c，如步骤2.a里面计算的那样。（回顾条目10，重要属性是指影响equals进行比较的属性）
2. 对于对象中剩下的每一个重要属性f，完成以下步骤：
    a. 为这个属性计算一个int型的哈希码c：
    i.  如果这个属性是基础类型的，计算Type.hashCode\(f\)，Type是f的类型对应的封箱基础类型。
    ii. 如果这个属性是个对象引用，而且这个对象所属类的equals方法通过递归调用equals方法来比较这个属性，则递归调用hashCode方法。如果需要更复杂的比较，则为这个属性计算一个“范式（canonical representation）”，然后针对这个范式调用hashCode方法。如果这个属性的值为null，则返回0（或者其它常量，但一般都是用0）。
    iii. 如果这个属性是个数组，则将它的每一个元素都当成重要属性来处理。也就是说，通过递归的应用这些规则来给每个元素计算出一个哈希码，然后按照步骤2.b将这些值合并起来。如果数组没有重要元素，则返回一个常量，最好不要用0。如果所有的元素都是重要的，则使用Arrays.hashCode。
    b. 将步骤2.a计算出来的哈希码c通过接下来的公式合并到result里去：result = 31\*result + c
3. 返回result。

When you are finished writing the hashCode method, ask yourself whether equal instances have equal hash codes. Write unit tests to verify your intuition \(unless you used AutoValue to generate your equals and hashCode methods, in which case you can safely omit these tests\). If equal instances have unequal hash codes, figure out why and fix the problem.

当我们完成hashCode方法的编写后，可以问下我们自己，是否相等的实例拥有相等的哈希码。我们可以写单元测试来验证下我们的判断（除非你需要Google的AutoValue来生成equals方法和hashCode方法，在这种情况下可以不用做这些测试）。如果相等的实例却没有相等的哈希码，查明原因后修复这个问题。

You may exclude derived fields from the hash code computation. In other words, you may ignore any field whose value can be computed from fields included in the computation.

在哈希码的计算过程中，我们可以将派生字段排除在外。换句话说，我们可以忽略一些属性，这些属性的值可以根据计算过程中所用到的属性计算出来。

You must exclude any fields that are not used in equals comparisons, or you risk violating the second provision of the hashCode contract.

我们一定要将equals比较过程中用不到的属性排除在外，不然将违反hashCode约定中的第二条。

The multiplication in step 2.b makes the result depend on the order of the fields, yielding a much better hash function if the class has multiple similar fields. For example, if the multiplication were omitted from a String hash function, all anagrams would have identical hash codes. The value 31 was chosen because it is an odd prime. If it were even and the multiplication overflowed, information would be lost, because multiplication by 2 is equivalent to shifting. The advantage of using a prime is less clear, but it is traditional. A nice property of 31 is that the multiplication can be replaced by a shift and a subtraction for better performance on some architectures: 31 \* i == \(i &lt;&lt; 5\) - i. Modern VMs do this sort of optimization automatically.

步骤2.b中的乘法部分使得result依赖于属性的顺序，如果类中有多个相似属性，则会产生更好的哈希函数。例如，如果String的哈希函数忽略了这个乘法，则所有只是字母顺序不同的字符串将会有相同的哈希码。我们之所以用31，是因为它是个奇素数。如果这个数是偶数并且乘法溢出的话，信息将被丢失，因为与2相乘其实是相当于位移。虽然使用素数的好处不是太明显，但这是一种习惯。31有个很好的特性，在一些体系结构中，可以通过位移和减法，31\*i == \(i&lt;&lt;5\) - i，来替代这个乘法，以取得更好的性能。现代的虚拟机都自动做了这类优化。

Let’s apply the previous recipe to the PhoneNumber class:

接下来让我们将前面讨论的方法用在PhoneNumber类上：

```
// Typical hashCode method
@Override 
public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```

Because this method returns the result of a simple deterministic computation whose only inputs are the three significant fields in a PhoneNumber instance, it is clear that equal PhoneNumber instances have equal hash codes. This method is, in fact, a perfectly good hashCode implementation for PhoneNumber, on par with those in the Java platform libraries. It is simple, is reasonably fast, and does a reasonable job of dispersing unequal phone numbers into different hash buckets.

在上面的代码里，hashCode用到的只是PhoneNumber实例的三个重要属性，并通过确定的计算返回了result。很明显，相等的PhoneNumber实例拥有相等的哈希码。事实上，PhoneNumber的这个hashCode方法的实现是很棒的，相当于Java类库的实现。这个方法简单，快，而且能把不相等的电话号码分布到不同的哈希桶里。

While the recipe in this item yields reasonably good hash functions, they are not state-of-the-art. They are comparable in quality to the hash functions found in the Java platform libraries’ value types and are adequate for most uses. If you have a bona fide need for hash functions less likely to produce collisions, see Guava’s com.google.common.hash.Hashing \[Guava\].

虽然依据本条目中的方法能生成合理的不错的哈希函数，但它们还不是最先进的。它们在质量上和Java类库的值类型的哈希函数相当，而且对大多数用途也足够了。如果你对哈希函数有真正的需求，希望其产生更少的冲突，请详阅Guava的com.google.common.hash.Hashing\[Guava\]。

The Objects class has a static method that takes an arbitrary number of objects and returns a hash code for them. This method, named hash, lets you write one-line hashCode methods whose quality is comparable to those written according to the recipe in this item. Unfortunately, they run more slowly because they entail array creation to pass a variable number of arguments, as well as boxing and unboxing if any of the arguments are of primitive type. This style of hash function is recommended for use only in situations where performance is not critical. Here is a hash function for PhoneNumber written using this technique:

Objects类里有一个静态方法，这个方法接受任意数量的object对象，然后返回一个哈希值。这个名叫hash的方法能让我们写出只有一行的hashCode方法，而且其质量和根据上述方法写出的方法相当。不幸的是，它们的运行得比较慢，因为它们需要创建数组来传入一个动态参数列表，如果这些参数里面还有基础类型，那还要做装箱和解装箱的动作。这种哈希函数建议仅用在对性能要求不高的场景。下面这个PhoneNumber的哈希函数就使用了这种技术：

```
// One-line hashCode method - mediocre performance
@Override 
public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
}
```

If a class is immutable and the cost of computing the hash code is significant, you might consider caching the hash code in the object rather than recalculating it each time it is requested. If you believe that most objects of this type will be used as hash keys, then you should calculate the hash code when the instance is created. Otherwise, you might choose to lazily initialize the hash code the first time hash-Code is invoked. Some care is required to ensure that the class remains thread-safe in the presence of a lazily initializedfield \(Item 83\). Our PhoneNumber class does not merit this treatment, but just to show you how it’s done, here it is. Note that the initial value for the hashCode field \(in this case, 0\) should not be the hash code of a commonly created instance:

如果一个类是不可变的，而且计算哈希码的代价很重要，我们也许应该考虑将哈希码缓存在对象里，而不是每次都去计算。如果你觉得某个类型的大多数对象将会被当做键来使用，那么你必须在实例被创建出来时就计算出哈希码。或者，你也可以选择延迟初始化（lazyily initialize）哈希码，即在hashCode方法第一次被调用时才进行生成动作。需要注意的是，在使用延迟初始化时，要确保类是线程安全的（条目83）。我们还不知道PhoneNumber类是否值得这么处理，但用来展示该如何实现。注意，hashCode属性的初始值（在这里是0）不应该是创建的实例的哈希码：

```
// hashCode method with lazily initialized cached hash code
private int hashCode; // Automatically initialized to 0
@Override 
public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    } 
    return result;
}
```

**Do not be tempted to exclude significant fields from the hash code computation to improve performance.** While the resulting hash function may run faster, its poor quality may degrade hash tables’ performance to the point where they become unusable. In particular, the hash function may be confronted with a large collection of instances that differ mainly in regions you’ve chosen to ignore. If this happens, the hash function will map all these instances to a few hash codes, and programs that should run in linear time will instead run in quadratic time.

**不要为了提高性能而试图在哈希码的计算过程中将重要的属性排除掉。**这样做虽然会让哈希函数运行得更快，但其随之变差的质量可能会导致哈希表慢到无法用。特别是，哈希函数可能会面临大量的实例，而这些实例主要的区别在于你选择忽视了的那些属性上。如果这种情况发生了，哈希函数将会把这些实例映射到少数的几个哈希码上，程序也将以线性时间运行，而不是平方时间。

This is not just a theoretical problem. Prior to Java 2, the String hash function used at most sixteen characters evenly spaced throughout the string, starting with the first character. For large collections of hierarchical names, such as URLs, this function displayed exactly the pathological behavior described earlier.

这不只是一个理论问题。在Java 2之前，String的哈希函数最多使用16个字符，从第一个字符开始，在整个字符串均匀地获取。对于层次状名字的大型集合，如URL，这个函数就产生了前面提到的病态行为。

**Don’t provide a detailed specification for the value returned by hashCode, so clients can’t reasonably depend on it; this gives you the flexibility to change it.** Many classes in the Java libraries, such as String and Integer, specify the exact value returned by their hashCode method as a function of the instance value. This is not a good idea but a mistake that we’re forced to live with: It impedes the ability to improve the hash function in future releases. If you leave the details unspecified and a flaw is found in the hash function or a better hash function is discovered, you can change it in a subsequent release.

**不要为hashCode返回的值提供详细的规范，这样的话客户端将不能合理地依赖它，而且不提供的话将能让你灵活地去做出改变。**Java类库里的很多类，如String和Integer，都把它们的hashCode方法返回的值指定为该实例值的一个函数。这并不是一个好主意，因为它强制我们必须去使用它：在未来的版本中，这也限制了它改进的能力。如果没有规定细节，而且当发现一个哈希函数的缺陷时，或者发现了一个更好的哈希函数时，我们就能在接下来的版本去做更改。

In summary, you must override hashCode every time you override equals, or your program will not run correctly. Your hashCode method must obey the general contract specified in Object and must do a reasonable job assigning unequal hash codes to unequal instances. This is easy to achieve, if slightly tedious, using the recipe on page 51. As mentioned in Item 10, the AutoValue framework provides a fine alternative to writing equals and hashCode methods manually, and IDEs also provide some of this functionality.

总结一句，每次我们覆盖equals方法时都必须覆盖hashCode方法，不然我们的程序将不会正常运作。我们的hashCode方法必须遵守Object里说明的通用约定，合理地为不相等的实例指派不相等的哈希码。我们按照前面说的方法就可以很容实现，虽然有点繁琐。就像条目10里提到的，AutoValue框架为手动编写equals方法和hashCode方法提供了一个不错的替换方式，多数IDE也提供了这个功能。

