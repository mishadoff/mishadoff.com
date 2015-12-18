---
layout: post
title: "Clojure Design Patterns"
date: 2015-12-18 11:34
comments: true
categories: [clojure, programming, java, story, patterns]
published: true
---

*Quick* overview of the classic [Design Patterns](http://en.wikipedia.org/wiki/Design_Patterns) in Clojure

<!-- more -->

**Disclaimer:** *Most patterns are easy to implement because
we use dynamic typing, functional programming and, of course,
Clojure. Some of them look wrong and ugly. It's okay.
All characters are fake, coincidences are accidental.*
	
### Index

- [Intro](#intro)
- [Episode  1. Command](#command)
- [Episode  2. Strategy](#strategy)
- [Episode  3. State](#state)
- [Episode  4. Visitor](#visitor)
- [Episode  5. Template Method](#template_method)
- [Episode  6. Iterator](#iterator)
- [Episode  7. Memento](#memento)
- [Episode  8. Prototype](#prototype)
- [Episode  9. Mediator](#mediator)
- [Episode 10. Observer](#observer)
- [Episode 11. Interpreter](#interpreter)
- [Episode 12. Flyweight](#flyweight)
- [Episode 13. Builder](#builder)
- [Episode 14. Facade](#facade)
- [Episode 15. Singleton](#singleton)
- [Episode 16. Chain of Responsibility](#chain) 
- [Episode 17. Composite](#composite) 
- [Episode 18. Factory Method](#factory_method)
- [Episode 19. Abstract Factory](#abstract_factory)
- [Episode 20. Adapter](#adapter)
- [Episode 21. Decorator](#decorator)
- [Episode 22. Proxy](#proxy)
- [Episode 23. Bridge](#bridge) 
- [Cheatsheet](#cheatsheet)
- [Cast](#cast)

### <div id="intro"/>Intro

> Our programming language is fucked up.  
> That's why we need design patterns.
>
> -- Anonymous

Two modest programmers **Pedro Veel** and **Eve Dopler**
are solving common software engineering problems and
applying design patterns.

### <div id="command"/>Episode 1. Command

> Leading IT service provider **"Serpent Hill & R.E.E"**
> acquired new project for USA customer.
> First delivery is a register, login and logout functionality
> for their brand new site.

**Pedro:** Oh, that's easy. You just need a Command interface...  

``` java
interface Command {
  void execute();
}
```

**Pedro:** Every action should implement this interface and define specific `execute` behaviour.

``` java
public class LoginCommand implements Command {

  private String user;
  private String password;

  public LoginCommand(String user, String password) {
    this.user = user;
    this.password = password;
  }

  @Override
  public void execute() {
    DB.login(user, password);
  }
}
```

``` java
public class LogoutCommand implements Command {

  private String user;

  public LogoutCommand(String user) {
    this.user = user;
  }

  @Override
  public void execute() {
    DB.logout(user);
  }
}
```

**Pedro:** Usage is simple as well.

``` java
(new LoginCommand("django", "unCh@1ned")).execute();
(new LogoutCommand("django")).execute();
```

**Pedro:** What do you think, Eve?  
**Eve:** Why are you using redundant wrapping into `LoginCommand` and just don't call `DB.login`?  
**Pedro:** It's important to wrap here, because now we can operate on generic `Command` objects.  
**Eve:** For what purpose?  
**Pedro:** Delayed call, logging, history tracking, caching, plenty of usages.  
**Eve:** Ok, how about that?  

``` clojure
(defn execute [command]
  (command))

(execute #(db/login "django" "unCh@1ned"))
(execute #(db/logout "django"))
```

**Pedro:** What the hell this hash sign?  
**Eve:** A shortcut for javaish  

``` java
new SomeInterfaceWithOneMethod() {
  @Override
  public void execute() {
    // do
  }
};
```

**Pedro:** Just like `Command` interface...  
**Eve:** Or if you want - no-hash solution.  

``` clojure
(defn execute [command & args]
  (apply command args))

(execute db/login "django" "unCh@1ned")
```

**Pedro:** And how do you save function for delayed call in that case?  
**Eve:** Answer yourself. What do you need to call a function?  
**Pedro:** Its name...  
**Eve:** And?  
**Pedro:** ...arguments.  
**Eve:** Bingo. All you do is saving a pair (*function-name*, *arguments*) and call it whenever you want using
`(apply function-name arguments)`  
**Pedro:** Hmm... Looks simple.  
**Eve:** Definitely, **Command is just a function.**  

### <div id="strategy"/>Episode 2. Strategy

> **Sven Tori** pays a lot of money
> to see a page with list of users.
> But users must be sorted by name and
> users with subscription must appear *before*
> all other users.
> Obviously, because they pay.
> Reverse sorting should keep subscripted users on top.

**Pedro:** Ha, just call `Collections.sort(users, comparator)`
with custom comparator.  
**Eve:** How would you implement *custom comparator*?  
**Pedro:** You need to take `Comparator` interface
and provide implementation for `compare(Object o1, Object o2)` method.
Also you need another implementation for `ReverseComparator`  
**Eve:** Stop talking, show me the code!  

``` java
class SubsComparator implements Comparator<User> {

  @Override
  public int compare(User u1, User u2) {
    if (u1.isSubscription() == u2.isSubscription()) {
      return u1.getName().compareTo(u2.getName());
    } else if (u1.isSubscription()) {
      return -1;
    } else {
      return 1;
    }
  }
}

class ReverseSubsComparator implements Comparator<User> {

  @Override
  public int compare(User u1, User u2) {
    if (u1.isSubscription() == u2.isSubscription()) {
      return u2.getName().compareTo(u1.getName());
    } else if (u1.isSubscription()) {
      return -1;
    } else {
      return 1;
    }
  }
}

// forward sort
Collections.sort(users, new SubsComparator());

// reverse sort
Collections.sort(users, new ReverseSubsComparator());
```

**Pedro:** Could you do the same?  
**Eve:** Yeah, something like that  

``` clojure
(sort (comparator 
       (fn [u1 u2]
         (cond
          (= (:subscription u1)
             (:subscription u2)) (neg? (compare (:name u1)
                                                (:name u2)))
             (:subscription u1) true
             :else false))) users)
```

**Pedro:** Pretty similar.  
**Eve:** But we can do it better  

``` clojure
;; forward sort
(sort-by (juxt (complement :subscription) :name) users)

;; reverse sort
(sort-by (juxt :subscription :name) #(compare %2 %1) users)
```

**Pedro:** Oh my gut! Monstrous oneliners.  
**Eve:** Functions, you know.  
**Pedro:** Whatever, it's very hard to understand what's happening there.

*Eve explains juxt, complement and sort-by*  
*10 minutes later*  

**Pedro:** Very doubtful approach to pass strategy.  
**Eve:** I don't care, because Strategy is just a **function passed to another function.**  

### <div id="state"/>Episode 3. State

> Sales person **Karmen Git**
> investigated the market and decided
> to provide user-specific functionality.

**Pedro:** Smooth requirements.  
**Eve:** Let's clarify them.  

- *If user has subscription show him all news in a feed*
- *Otherwise, show him only recent 10 news*
- *If he pays money, add the amount to his account balance*
- *If user doesn't have subscription and there is enough money
to buy subscription, change his state to...*

**Pedro:** State! Awesome pattern. First we make a user state enum  

``` java
public enum UserState {
  SUBSCRIPTION(Integer.MAX_VALUE),
  NO_SUBSCRIPTION(10);

  private int newsLimit;

  UserState(int newsLimit) {
    this.newsLimit = newsLimit;
  }

  public int getNewsLimit() {
    return newsLimit;
  }
}
```

**Pedro:** User logic is following  

``` java
public class User {
  private int money = 0;
  private UserState state = UserState.NO_SUBSCRIPTION;
  private final static int SUBSCRIPTION_COST = 30;

  public List<News> newsFeed() {
    return DB.getNews(state.getNewsLimit());
  }

  public void pay(int money) {
    this.money += money;
    if (state == UserState.NO_SUBSCRIPTION
	    && this.money >= SUBSCRIPTION_COST) {
      // buy subscription
      state = UserState.SUBSCRIPTION;
      this.money -= SUBSCRIPTION_COST;
    }
  }
}
```

**Pedro:** Lets call it

``` java
User user = new User(); // create default user
user.newsFeed(); // show him top 10 news
user.pay(10); // balance changed, not enough for subs
user.newsFeed(); // still top 10
user.pay(25); // balance enough to apply subscription
user.newsFeed(); // show him all news
```

**Eve:** You just hide value that affects behaviour inside `User` object. We could use strategy to pass it directly `user.newsFeed(subscriptionType)`.  
**Pedro:** Agreed, State is very close to the Strategy.
They even have the same UML diagrams. but we encapsulate balance and bind it to user.  
**Eve:** I think it achieves the same goal using another mechanism. Instead of providing strategy explicitly, it depends on some state. From clojure perspective it can be implemented
the same way as strategy pattern.  
**Pedro:** But successive calls can change object's state.  
**Eve:** Correct, but it has nothing to do with `Strategy` it is just implementation detail.  
**Pedro:** What about "another mechanism"?  
**Eve:** Multimethods.  
**Pedro:** Multi *what*?  
**Eve:** Look at this  

``` clojure
(defmulti news-feed :user-state)

(defmethod news-feed :subscription [user]
  (db/news-feed))

(defmethod news-feed :no-subscription [user]
  (take 10 (db/news-feed)))
```

**Eve:** And `pay` function it's just a plain function, which changes state of object. We don't like state too much in clojure, but if you wish.  

``` clojure
(def user (atom {:name "Jackie Brown"
                 :balance 0
                 :user-state :no-subscription}))
				 
(def ^:const SUBSCRIPTION_COST 30)

(defn pay [user amount]
  (swap! user update-in [:balance] + amount)
  (when (and (>= (:balance @user) SUBSCRIPTION_COST)
             (= :no-subscription (:user-state @user)))
    (swap! user assoc :user-state :subscription)
    (swap! user update-in [:balance] - SUBSCRIPTION_COST)))

(news-feed @user) ;; top 10
(pay user 10)
(news-feed @user) ;; top 10
(pay user 25)
(news-feed @user) ;; all news
```

**Pedro:** Is dispatching by multimethods better than dispatching by enum?  
**Eve:** No, in this particlular case, but in general yes.  
**Pedro:** Explain, please  
**Eve:** Do you know what *double dispatch* is?  
**Pedro:** Not sure.  
**Eve:** Well, it is topic for `Visitor` pattern.

### <div id="visitor"/>Episode 4. Visitor

> **Natanius S. Selbys** suggested to implement functionality
> which allows users export their messages, activities
> and achievements in different formats.

**Eve:** So, how do you plan to do it?  
**Pedro:** We have one hierarchy for item types
(Message, Activity) and another
for file formats (PDF, XML)  

``` java
abstract class Format { }
class PDF extends Format { }
class XML extends Format { }

public abstract class Item {
  void export(Format f) {
    throw new UnknownFormatException(f);
  }
  abstract void export(PDF pdf);
  abstract void export(XML xml);
}

class Message extends Item {
  @Override
  void export(PDF f) {
    PDFExporter.export(this);
  }

  @Override
  void export(XML xml) {
    XMLExporter.export(this);
  }
}

class Activity extends Item {
  @Override
  void export(PDF pdf) {
    PDFExporter.export(this);
  }

  @Override
  void export(XML xml) {
    XMLExporter.export(this);
  }
}
```

**Pedro:** That's all.  
**Eve:** Nice, but how do you dispatch on argument type?  
**Pedro:** What the problem?  
**Eve:** Consider this snippet  

``` java
Item i = new Activity();
Format f = new PDF();
i.export(f);
```

**Pedro:** Nothing suspicious here.  
**Eve:** Actually, if you run this code you get `UnknownFormatException`  
**Pedro:** Wait...Really?  
**Eve:** In java you can use only *single dispatch*. That means if you call `i.export(f)`
you dispatches on the actual type of `i`, not `f`.  
**Pedro:** I'm surprised. So, there is no dispatch on argument type?  
**Eve:** That's what visitor hack for. After you got a dispatch on `i` type, you
additionally call `f.someMethod(i)` and dispatched on `f` type.  
**Pedro:** How that looks in code?  
**Eve:** You separately define export operations for all types as a `Visitor`  

``` java
public interface Visitor {
  void visit(Activity a);
  void visit(Message m);
}

public class PDFVisitor implements Visitor {
  @Override
  public void visit(Activity a) {
    PDFExporter.export(a);
  }

  @Override
  public void visit(Message m) {
    PDFExporter.export(m);
  }
}
```

**Eve:** Your items change signature to accept different visitors.  

``` java
public abstract class Item {
  abstract void accept(Visitor v);
}

class Message extends Item {
  @Override
  void accept(Visitor v) {
    v.visit(this);
  }
}

class Activity extends Item {
  @Override
  void accept(Visitor v) {
    v.visit(this);
  }
}
```

**Eve:** To use it you may call  

``` java
Item i = new Message();
Visitor v = new PDFVisitor(); 
i.accept(v);
```

**Eve:** And everything works fine.
Moreover, you can add new operations for
activities and messages
by just defining new visitors and
without changing their code.  
**Pedro:** That's really useful. But implementation is tough,
it is the same for clojure?  
**Eve:** Not really, clojure supports it natively via multimethods  
**Pedro:** Multi *what*?  
**Eve:** Just follow the code...
First we define dispatcher *function*

``` clojure
(defmulti export
  (fn [item format] [(:type item) format]))
```

**Eve:** It accepts `item` and `format` to be exported. Examples:

``` clojure
;; Message
{:type :message :content "Say what again!"}
;; Activity
{:type :activity :content "Quoting Ezekiel 25:17"}
;; Formats
:pdf, :xml
```

**Eve:** And now you just provide a functions
for different combinatations, and dispatcher decide which one to call.

``` clojure
(defmethod export [:activity :pdf] [item format]
  (exporter/activity->pdf item))

(defmethod export [:activity :xml] [item format]
  (exporter/activity->xml item))

(defmethod export [:message :pdf] [item format]
  (exporter/message->pdf item))

(defmethod export [:message :xml] [item format]
  (exporter/message->xml item))
```

**Pedro:** What if unknown format passed?  
**Eve:** We could specify default dipatcher function.  

``` clojure
(defmethod export :default [item format]
  (throw (IllegalArgumentException. "not supported")))

```

**Pedro:** Ok, but there is no hierarchy for `:pdf` and `:xml`.
They are just keywords?  
**Eve:** Correct, simple problem - simple solution.
If you need advanced features, you could use *adhoc hierarchies*
or dispatch by `class`.  

``` clojure
(derive ::pdf ::format)
(derive ::xml ::format)
```

**Pedro:** Quadrocolons?!  
**Eve:** Assume they are just keywords.  
**Pedro:** Ok.  
**Eve:** Then you add functions for every dispatch type
`::pdf`, `::xml` and `::format`  

``` clojure
(defmethod export [:activity ::pdf])
(defmethod export [:activity ::xml])
(defmethod export [:activity ::format])
```

**Eve:** If some new format (i.e. csv) appears in the system  

``` clojure
(derive ::csv ::format)
```

**Eve:** It will be dispatched to `::format` function,
until you add a separate `::csv` function.  
**Pedro:** Seems good.  
**Eve:** Definitely, much easier.  
**Pedro:** So, basically, **if a language support multiple dispatch,
you don't need Visitor pattern**?  
**Eve:** Exactly.  

### <div id="template_method"/>Episode 5. Template Method

> MMORPG **Mech Dominore Fight Saga** requested to implement
> a game bot for their VIP users. Not fair.

**Pedro:** First, we must decide what actions 
should be automated with bot.  
**Eve:** Have you ever played RPG?  
**Pedro:** Fortunately, no  
**Eve:** Oh my... Let's go, I'll show you...  

*2 weeks later*

**Pedro:** ...fantastic, I found epic sword, which has +100 attack.  
**Eve:** Unbelievable. But now, it's time for bot.  
**Pedro:** Easy-peasy. We could select following events  

- Battle
- Quest
- Open Chest

**Pedro:** Characters behave differently in
different events, for example mages cast spells in battle,
but rogues prefer silent melee combat; locked chests are skipped
by most characters, but rogues can unlock them, etc.  
**Eve:** Looks like ideal candidate for `Template Method`?  
**Pedro:** Yes. We define abstract algorithm, and then specify
differences in subclasses.  

``` java
public abstract class Character {
  void moveTo(Location loc) {
    if (loc.isQuestAvailable()) {
      Journal.addQuest(loc.getQuest());
    } else if (loc.containsChest()) {
      handleChest(loc.getChest());
    } else if (loc.hasEnemies()) {
      attack(loc.getEnemies());
    }
    moveTo(loc.getNextLocation());
  }

  private void handleChest(Chest chest) {
    if (!chest.isLocked()) {
      chest.open();
    } else {
      handleLockedChest(chest);
    }
  }

  abstract void handleLockedChest(Chest chest);
  abstract void attack(List<Enemy> enemies);
}
```

**Pedro:** We've separated to `Character` class everything common to all characters.
Now we can create subclasses, that define how character should behave in specific situation.
In out case: *handling locked chests* and *attacking enemies*.  
**Eve:** Let's start with a Mage class.  
**Pedro:** Mage? Okay.
He can't open locked chest, so implementation is just *do nothing*.
And if he is attacking enemies, if there are more than 10 enemies,
freeze them, and cast teleport to run away. If there are 10 enemies or less
cast fireball on each of them.  

``` java
public class MageCharacter extends Character {
  @Override
  void handleLockedChest(Chest chest) {
    // do nothing
  }

  @Override
  void attack(List<Enemy> enemies) {
    if (enemies.size() > 10) {
      castSpell("Freeze Nova");
      castSpell("Teleport");
    } else {
      for (Enemy e : enemies) {
        castSpell("Fireball", e);
      }
    }
  }
}
```

**Eve:** Excellent, what about Rogue class?  
**Pedro:** Easy as well, rogues can unlock chests and
prefer silent combat, handle enemies one by one.  

``` java
public class RogueCharacter extends Character {
  @Override
  void handleLockedChest(Chest chest) {
    chest.unlock();
  }

  @Override
  void attack(List<Enemy> enemies) {
    for (Enemy e : enemies) {
      invisibility();
	  attack("backstab", e);
    }
  }
}
```

**Eve:** Excellent. But how this approach is differrent from Strategy?  
**Pedro:** What?  
**Eve:** I mean, you redefined behaviour by using subclasses,
but in Strategy pattern you did the same:
redefined behaviour by using functions.  
**Pedro:** Well, another approach.  
**Eve:** State was handled with another approach as well.  
**Pedro:** What are you trying to say?  
**Eve:** You are solving the same kind of problem,
but change the approach to it.  
**Pedro:** How do yo solve this problem using strategy in clojure?  
**Eve:** Just pass a set of specific functions for each character. For example, your abstract move may look like:  

``` clojure
(defn move-to [character location]
  (cond
   (quest? location)
   (journal/add-quest (:quest location))

   (chest? location)
   (handle-chest (:chest location))

   (enemies? location)
   (attack (:enemies location)))
  (move-to character (:next-location location)))
```

**Eve:** To add character-specific implementation
of methods `handle-chest` and `attack`, implement them
and pass as an argument.  

``` clojure
;; Mage-specific actions
(defn mage-handle-chest [chest])

(defn mage-attack [enemies]
  (if (> (count enemies) 10)
    (do (cast-spell "Freeze Nova")
        (cast-spell "Teleport"))
    ;; otherwise
    (doseq [e enemies]
      (cast-spell "Fireball" e))))

;; Signature of move-to will change to

(defn move-to [character location
               & {:keys [handle-chest attack]
                  :or {handle-chest (fn [chest])
                       attack (fn [enemies] (run-away))}}]
  ;; previous implementation
)
```

**Pedro:** OMG, what's happening there?  
**Eve:** We changed signature of `move-to` to accept
`handle-chest` and `attack` functions.  
Think of them like optional parameters.  

``` clojure
(move-to character location
  :handle-chest mage-handle-chest
  :attack       mage-attack)
```

**Eve:** Keep in mind that if these functions are not provided
we use default behavior: do nothing for `handle-chest` and
run away from enemies in `attack`  
**Pedro:** Fine, but is this better than approach by subclassing? Seems that we have a lot of redundant information in
`move-to` call.  
**Eve:** It's fixable, just define this call once, and give it
alias  

``` clojure
(defn mage-move [character location]
  (move-to character location
    :handle-chest mage-handle-chest
    :attack       mage-attack))
```

**Eve:** Or use multimethods, it's even better.  

``` clojure
(defmulti move 
  (fn [character location] (:class character)))

(defmethod move :mage [character location]
  (move-to character location
    :handle-chest mage-handle-chest
    :attack       mage-attack))
```

**Pedro:** I understand. But why do you think pass as argument is better than subclassing?  
**Eve:** You can change behaviour dynamically. Assume your mage
has no mana, so instead of trying to cast fireballs, he can just teleport and run away, you just provide new function.  
**Pedro:** Makes sense. **Functions everywhere**.  

### <div id="iterator"/>Episode 6. Iterator

> Technical consultant **Kent Podiololis**
> complains for C-style loops usage.
>
> "Are we in 1980 or what?" -- Kent

**Pedro:** We definitely should use pattern Iterator from java.  
**Eve:** Don't be fool, nobody's using `java.util.Iterator`  
**Pedro:** Everybody use it implicitly in `for-each` loop. It's a good way to traverse a container.  
**Eve:** What does it mean *"to traverse a container"*?  
**Pedro:** Formally, the container should provide two methods for you:  
`next()` to return next element and `hasNext()` to return true if container has more elements.  
**Eve:** Ok. Do you know what linked list is?  
**Pedro:** Singly linked list?  
**Eve:** Singly linked list.  
**Pedro:** Sure. It is a container consists of nodes. 
Each node has a data value and reference to the next node.
And `null` value if there is no next node.  
**Eve:** Correct. Now tell me how traversing such list is differ
from traversing via iterator?  
**Pedro:** Emmm...  

Pedro wrote two traversing snippets:

* Traversing using iterator

``` java
Iterator i;
while (i.hasNext()) {
  i.next();
}
```

* Traversing using linked list

``` java
Node next = root;
while (next != null) {
  next = next.next;
}
```

**Pedro:** They are pretty similar...What is analogue of `Iterator` in clojure?  
**Eve:** `seq` function.  


``` clojure
(seq [1 2 3])       => (1 2 3)
(seq (list 4 5 6))  => (4 5 6)
(seq #{7 8 9})      => (7 8 9)
(seq (int-array 3)) => (0 0 0)
(seq "abc")         => (\a \b \c)
```

**Pedro:** It returns a list...  
**Eve:** Sequence, because **Iterator is just a sequence**  
**Pedro:** Is it possible to make `seq` works on custom datastructures?  
**Eve:** Implement `clojure.lang.Seqable` interface  

``` clojure
(deftype RedGreenBlackTree [& elems]
  clojure.lang.Seqable
  (seq [self]
	;; traverse element in needed order
	))
```

**Pedro:** Fine then. But I've heard iterator is often used to achive *laziness*, for example
to calculate value only during `getNext()` call, how list handle that?  
**Eve:** List can be lazy as well, clojure calls such list *"lazy sequence"*.  

``` clojure
(def natural-numbers (iterate inc 1))
```

**Eve:** We defined thing to represent *ALL* natural numbers, but we haven't got `OutOfMemory` yet,
because we haven't requested any value. It's lazy.  
**Pedro:** Could you explain more?  
**Eve:** Unfortunately, I am too lazy for that.  
**Pedro:** I will remember that!  

### <div id="memento"/>Episode 7: Memento

> User **Chad Bogue** lost the message he was writing
> for two days. Implement save button for him.

**Pedro:** I don't believe there are people who
can type in textbox for two days. Two. Days.  
**Eve:** Let's *save* him.  
**Pedro:** I *googled* this problem. Most popular approach
to implement save button is Memento pattern.
You need *originator*, *caretaker* and *memento* objects.  
**Eve:** What's that?  
**Pedro:** *Originator* is just an object or state that we want to preserve.
(text inside a textbox), *caretaker* is responsible to save state (save button)
and *memento* is just an object to encapsulate state.  

``` java
public class TextBox {
  // state for memento
  private String text = "";

  // state not handled by memento
  private int width = 100;
  private Color textColor = Color.BLACK;

  public void type(String s) {
    text += s;
  }

  public Memento save() {
    return new Memento(text);
  }

  public void restore(Memento m) {
    this.text = m.getText();
  }

  @Override
  public String toString() {
    return "[" + text + "]";
  }
}
```

**Pedro:** Memento is just an immutable object  

``` java
public final class Memento {
  private final String text;

  public Memento(String text) {
    this.text = text;
  }

  public String getText() {
    return text;
  }
}
```

**Pedro:** And caretaker is a demo  

``` java
// open browser, init empty textbox
TextBox textbox = new TextBox();

// type something into it
textbox.type("Dear, Madonna\n");
textbox.type("Let me tell you what ");

// press button save
Memento checkpoint1 = textbox.save();

// type again
textbox.type("song 'Like A Virgin' is about. ");
textbox.type("It's all about a girl...");

// suddenly browser crashed, restart it, reinit textbox
textbox = new TextBox();

// but it's empty! All work is gone!
// not really, you rollback to last checkpoint
textbox.restore(checkpoint1);
```

**Pedro:** Just a note if you want a multiple checkpoints, save memento's to the list.  
**Eve:** *Originator*, *caretaker*, *memento* - looks as a bunch of nouns, but actually it's all about two functions `save` and `restore`.  

``` clojure
(def textbox (atom {}))

(defn init-textbox [] 
 (reset! textbox {:text ""
                  :color :BLACK
                  :width 100}))

(def memento (atom nil))

(defn type-text [text]
  (swap! textbox
    (fn [m]
      (update-in m [:text] (fn [s] (str s text))))))

(defn save []
  (reset! memento (:text @textbox)))

(defn restore []
  (swap! textbox assoc :text @memento))
```

**Eve:** And demo as well.

``` clojure
(init-textbox)
(type-text "'Like A Virgin' ")
(type-text "it's not about this sensitive girl ")
(save)
(type-text "who meets nice fella")
;; crash
(init-textbox)
(restore)
```

**Pedro:** It's pretty the same code.  
**Eve:** Yes, but you must care about memento immutability  
**Pedro:** What does it mean?  
**Eve:** You are lucky, that you got `String` object in this example,
`String` is immutable. But if you have something, that
may change its internal state, you need to perform deep copy of this object for memento.  
**Pedro:** Oh, right. It's just a recursive `clone()` calls to
obtain prototype.  
**Eve:** We will talk about Prototype in a minute, but just
remember that `Memento` is not about *caretaker* and *originator*, it is about **save** and **restore**.  

### <div id="prototype"/>Episode 8: Prototype

> **Dex Ringeus** detected that users feel uncomfortable
> with registration form. Make it more usable.

**Pedro:** So, what's the problem with the registration?  
**Eve:** There are lot of fields users bored to type in.  
**Pedro:** For example?  
**Eve:** For example, `weight`. Having such field scares
90% of female users.  
**Pedro:** But this field is important for our analytics
system, we make food and clothes recomendations based on that field.  
**Eve:** Then, make it optional, and if it is not provided, take
some default value.  
**Pedro:** `60 kg` is ok?  
**Eve:** I think so.  
**Pedro:** Ok, give me two minutes.  

*2 hours later*

**Pedro:** I suggest to use some registration *prototype*
which has all fields are filled with default values. 
After user completes the form we modify filled values.  
**Eve:** Sounds great.  
**Pedro:** Here it is our standard registration form, with prototype in `clone()` method.  

``` java
public class RegistrationForm implements Cloneable {
  private String name = "Zed";
  private String email = "zzzed@gmail.com";
  private Date dateOfBirth = new Date(1970, 1, 1);
  private int weight = 60;
  private Gender gender = Gender.MALE;
  private Status status = Status.SINGLE;
  private List<Child> children = Arrays.asList(new Child(Gender.FEMALE));
  private double monthSalary = 1000;
  private List<Brand> favouriteBrands = Arrays.asList("Adidas", "GAP");
  // few hundreds more properties

  @Override
  protected RegistrationForm clone() throws CloneNotSupportedException {
    RegistrationForm prototyped = new RegistrationForm();
      prototyped.name = name;
      prototyped.email = email;
      prototyped.dateOfBirth = (Date)dateOfBirth.clone();
      prototyped.weight = weight;
      prototyped.status = status;
      List<Child> childrenCopy = new ArrayList<Child>();
      for (Child c : children) {
        childrenCopy.add(c.clone());
      }
      prototyped.children = childrenCopy;
      prototyped.monthSalary = monthSalary;
      List<String> brandsCopy = new ArrayList<String>();
      for (String s : favouriteBrands) {
        brandsCopy.add(s);
      }
      prototyped.favouriteBrands = brandsCopy;
    return  prototyped;
  }
}
```

**Pedro:** Every time we create a user, call `clone()` and then override needed properties.  
**Eve:** Awful! In mutable world `clone()` is needed to create
new object with the same properties.
The hard part is the copy must be deep, i.e. instead of copying
reference you need recursively `clone()` other objects,
and what if one of them doesn't have `clone()`...  
**Pedro:** That's the problem and this pattern solves it.  
**Eve:** I don't think it is a solution if you need to implement
clone every time you adding new object.  
**Pedro:** How clojure avoid this?  
**Eve:** Clojure has immutable data structures. That's all.  
**Pedro:** How does it solve prototype problem?  
**Eve:** Every time you modify object, you get
a fresh new immutable copy of your data, and old one is not changed. **Prototype is not needed in immutable world**  

``` clojure
(def registration-prototype
	 {:name          "Zed"
	  :email         "zzzed@gmail.com"
	  :date-of-birth "1970-01-01"
	  :weight        60
	  :gender        :male
	  :status        :single
	  :children      [{:gender :female}]
	  :month-salary  1000
	  :brands        ["Adidas" "GAP"]})

;; return new object
(assoc registration-prototype 
	 :name "Mia Vallace"
	 :email "tomato@gmail.com"
	 :weight 52
	 :gender :female
	 :month-salary 0)
```

**Pedro:** Great! But how this affects performance?  
Copying million rows each time you adding new value seems
time consuming operation.  
**Eve:** No, it is not. Go to Google and search for *persistent data structures* and *structural sharing*  
**Pedro:** Thanks a lot.  

### <div id="mediator"/>Episode 9: Mediator

> Recently performed external code review
> shows a lot of issues with current codebase.
> **Veerco Wierde** emphasizes tight coupling 
> in chat application.

**Eve:** What is tight coupling?  
**Pedro:** It's the problem when objects
know too much about each other.  
**Eve:** Could you be more specific?  
**Pedro:** Look at the current chat implementation  

``` java
public class User {
  private String name;
  List<User> users = new ArrayList<User>();

  public User(String name) {
    this.name = name;
  }

  public void addUser(User u) {
    users.add(u);
  }

  void sendMessage(String message) {
    String text = String.format("%s: %s\n", name, message);
    for (User u : users) {
      u.receive(text);
    }
  }

  private void receive(String message) {
    // process message
  }
}
```

**Pedro:** The problem here is the user knows
everything about other users. It is very hard to use and maintain such code.
When new user connects to the chat, you must add a reference to him via `addUser`
for every existing user.  
**Eve:** So, we just move one piece of responsibility to another class?  
**Pedro:** Yes, kind of. We create *mega-aware* class, called mediator, that binds
all parts together. Obviously, each part knows only about mediator.  

``` java
public class User {
  String name;
  private Mediator m;

  public User(String name, Mediator m) {
    this.name = name;
    this.m = m;
  }

  public void sendMessage(String text) {
    m.sendMessage(this, text);
  }

  public void receive(String text) {
    // process message
  }
}

public class Mediator {

  List<User> users = new ArrayList<User>();

  public void addUser(User u) {
    users.add(u);
  }

  public void sendMessage(User u, String text) {
    for (User user : users) {
      u.receive(text);
    }
  }
}
```

**Eve:** That seems like a simple refactoring problem.  
**Pedro:** Profit may be underestimated, but
if you have hundreds of mutually connected components
(UI for example) mediator is really a savior.  
**Eve:** Agreed.  
**Pedro:** Now the clojure turn.  
**Eve:** Ok...let's look...your mediator is responsible
for *saving users* and *sending messages*  

``` clojure
(def mediator
  (atom {:users []
         :send (fn [users text]
			     (map #(receive % text) users))}))

(defn add-user [u]
	(swap! mediator 
	  (fn [m]
	    (update-in m [:users] conj u))))

(defn send-message [u text]
	(let [send-fn (:send @mediator)
	      users (:users @mediator)]
	  (send-fn users (format "%s: %s\n" (:name u) text))))

(add-user {:name "Mister White"})
(add-user {:name "Mister Pink"})
(send-message {:name "Joe"} "Toby?")
```

**Pedro:** Good enough.  
**Eve:** Nothing interesing here, because it is just a one approach **to reduce coupling**.

### <div id="observer"/>Episode 10: Observer

> Independent security commision
> detected hacker **Dartee Hebl** has over a billion
> dollars balance on his account.
> Track big transactions to the accounts.

**Pedro:** Are we *Holmses*?  
**Eve:** No, but there is a lack of logging in the system,
need to find a way to track all changes to balance.  
**Pedro:** We could add observers. Every time we modify
balance, if change is *big enough*, notify about this and track the reason. We need `Observer` interface  

``` java
public interface Observer {
  void notify(User u);
}
```

**Pedro:** And two specific observers  

``` java
class MailObserver implements Observer {
  @Override
  public void notify(User user) {
    MailService.sendToFBI(user);
  }
}

class BlockObserver implements Observer {
  @Override
  public void notify(User u) {
    DB.blockUser(u);
  }
}
```

**Pedro:** `Tracker` class would be responsible
for managing observers.  

``` java
public class Tracker {
  private Set<Observer> observers = new HashSet<Observer>();

  public void add(Observer o) {
    observers.add(o);
  }

  public void update(User u) {
    for (Observer o : observers) {
      o.notify(u);
    }
  }
}
```

**Pedro:** And the last part: init tracker with the user and
modify its `addMoney` method. If transcation amount is greaterthan `100$`, notify FBI and block this user.  

``` java
public class User {
  String name;
  double balance;
  Tracker tracker;

  public User() {
    initTracker();
  }

  private void initTracker() {
    tracker = new Tracker();
    tracker.add(new MailObserver());
    tracker.add(new BlockObserver());
  }


  public void addMoney(double amount) {
    balance += amount;
    if (amount > 100) {
      tracker.update(this);
    }
  }
}
```

**Eve:** Why are you created two separate observers?
You could use it in a one.  

``` java
class MailAndBlock implements Observer {
  @Override
  public void notify(User u) {
    MailService.sendToFBI(u);
    DB.blockUser(u);
  }
}
```

**Pedro:** Single responsibility principle.  
**Eve:** Oh, yeah.  
**Pedro:** And you can compose observer functionality
dynamically.  
**Eve:** I see your point.  

``` clojure
;; Tracker

(def observers (atom #{}))

(defn add [observer]
  (swap! observers conj observer))

(defn notify [user]
  (map #(apply % user) @observers))

;; Fill Observers

(add (fn [u] (mail-service/send-to-fbi u)))
(add (fn [u] (db/block-user u)))

;; User

(defn add-money [user amount]
  (swap! user
    (fn [m]
      (update-in m [:balance] + amount)))
  ;; tracking
  (if (> amount 100) (notify)))
```

**Pedro:** It's a pretty the same way?  
**Eve:** Yeah, in fact observer is just a way to register **function, which will be called after another function**.  
**Pedro:** It is still the pattern.  
**Eve** Sure, but we can improve solution a bit using clojure watches.

``` clojure
(add-watch
  user
  :money-tracker 
  (fn [k r os ns] 
    (if (< 100 (- (:balance ns) (:balance os)))
      (notify))))
```

**Pedro:** Why is that better?  
**Eve:** First of all, our `add-money` function is clean, it just adds money. Also, watcher tracks *every* change to the state, not the ones we handle in mutator functions, like `add-money`  
**Pedro:** Explain please.  
**Eve:** If there is provided another secred method `secret-add-money` for changing balance, watchers will handle that as well.  
**Pedro:** That's awesome!  

### <div id="interpreter"/>Episode 11: Interpreter

> **Bertie Prayc** stole important data from our server
> and shared it via BitTorrent system.
> Create a fake account for *Bertie* to discredit him.

**Pedro:** BitTorrent system based on `.torrent` files.
We need to write [Bencode](http://en.wikipedia.org/wiki/Bencode) encoder.  
**Eve:** Yes, but first let's agree on the format spec  

Bencode encoding rules:

- Two datatypes are supported:
  - Integer `N` is encoded as `i<N>e`. (42 = i42e)
  - String `S` is encoded as `<length>:<contents>` (hello = 5:hello)
- Two containers are supported:
  - List of values is encoded as `l<contents>e` ([1, "Bye"] = li1e3:Byee)
  - Map of values is encoded as `d<contents>e` ({"R" 2, "D" 2} = d1:Ri2e1:Di2ee)
     - Keys are just strings, Values are any bencode element

**Pedro:** Seems easy.  
**Eve:** Maybe, but take into account that values may be nested, list inside list, etc.  
**Pedro:** Sure. I think we can use *Interpreter* pattern for bencode encoding.  
**Eve:** Try it.  
**Pedro:** We start from interface for all bencode elements  

``` java
interface BencodeElement {
  String interpret();
}
```

**Pedro:** Then we provide implementation for each datatype and datacontainer  

``` java
class IntegerElement implements BencodeElement {
  private int value;

  public IntegerElement(int value) {
    this.value = value;
  }

  @Override
  public String interpret() {
    return "i" + value + "e";
  }
}

class StringElement implements BencodeElement {
  private String value;

  StringElement(String value) {
    this.value = value;
  }

  @Override
  public String interpret() {
    return value.length() + ":" + value;
  }
}

class ListElement implements BencodeElement {
  private List<? extends BencodeElement> list;

  ListElement(List<? extends BencodeElement> list) {
    this.list = list;
  }

  @Override
  public String interpret() {
    String content = "";
    for (BencodeElement e : list) {
      content += e.interpret();
    }
    return "l" + content + "e";
  }
}

class DictionaryElement implements BencodeElement {
  private Map<StringElement, BencodeElement> map;

  DictionaryElement(Map<StringElement, BencodeElement> map) {
    this.map = map;
  }

  @Override
  public String interpret() {
    String content = "";
    for (Map.Entry<StringElement, BencodeElement> kv : map.entrySet()) {
      content += kv.getKey().interpret() + kv.getValue().interpret();
    }
    return "d" + content + "e";
  }
}
```

**Pedro:** And finally, our bencoded string can be constructed from common datastructures programmatically

``` java
// discredit user
Map<StringElement, BencodeElement> mainStructure = new HashMap<StringElement, BencodeElement>();
// our victim
mainStructure.put(new StringElement("user"), new StringElement("Bertie"));
// just downloads files
mainStructure.put(new StringElement("number_of_downloaded_torrents"), new IntegerElement(623));
// and nothing uploads
mainStructure.put(new StringElement("number_of_uploaded_torrents"), new IntegerElement(0));
// and nothing donates
mainStructure.put(new StringElement("donation_in_dollars"), new IntegerElement(0));
// prefer dirty categories
mainStructure.put(new StringElement("preffered_categories"),
                      new ListElement(Arrays.asList(
                          new StringElement("porn"),
                          new StringElement("murder"),
                          new StringElement("scala"),
                          new StringElement("pokemons")
                      )));
BencodeElement top = new DictionaryElement(mainStructure);

// let's totally discredit him
String bencodedString = top.interpret();
BitTorrent.send(bencodedString);
```

**Eve:** Interesting, but that is ton of code!  
**Pedro:** We pay readability for capabilities.  
**Eve:** I suppose you've heard concept **Code is Data**, that's a lot easier in clojure  

``` clojure
;; multimethod to handle bencode structure
(defmulti interpret class)

;; implementation of bencode handler for each type
(defmethod interpret java.lang.Long [n]
  (str "i" n "e"))

(defmethod interpret java.lang.String [s]
  (str (count s) ":" s))

(defmethod interpret clojure.lang.PersistentVector [v]
  (str "l" 
       (apply str (map interpret v))
	   "e"))

(defmethod interpret clojure.lang.PersistentArrayMap [m]
  (str "d" 
       (apply str (map (fn [[k v]] 
	                     (str (interpret k)
                              (interpret v))) m))
       "e"))

;; usage
(interpret {"user" "Bertie"
            "number_of_downloaded_torrents" 623
			"number_of_uploaded_torrent" 0
			"donation_in_dollars" 0
			"preffered_categories" ["porn"
                                    "murder"
			                        "scala"
									"pokemons"]})
```

**Eve:** You see how it is much easier define specific data?  
**Pedro:** Sure, and `interpret` it's just a **function** per bencode type, instead of separate class.  
**Eve:** Correct, interpreter is nothing but a **set of functions to process a tree**.

### <div id="flyweight"/>Episode 12: Flyweight

> Administrators of the lawyer firm **Cristopher, Matton & Pharts**
> detected that reporting system consumes a lot of memory
> and garbage collector continiously hangs system for seconds. Fix that.

**Pedro:** I've seen this issue before.  
**Eve:** What's wrong?  
**Pedro:** They have realtime charts with a lot of different points.
It's a huge amount of memory. As a result garbage collector stops the system.  
**Eve:** Hmmm, what we can do?  
**Pedro:** Not much, caching does not help us because points are...  
**Eve:** Wait!  
**Pedro:** What?  
**Eve:** They use age values for points, why not precompute these points for most common ages?
Say for age [0, 100]  
**Pedro:** You mean use *Flyweight* pattern?  
**Eve:** I mean reuse objects.  

``` java
class Point {
  int x;
  int y;

  /* some other properties*/

  // precompute 10000 point values at class loading time
  private static Point[][] CACHED;
  static {
    CACHED = new Point[100][];
    for (int i = 0; i < 100; i++) {
      CACHED[i] = new Point[100];
      for (int j = 0; j < 100; j++) {
        CACHED[i][j] = new Point(i, j);
      }
    }
  }

  Point(int x, int y) {
    this.x = x;
    this.y = y;
  }

  static Point makePoint(int x, int y) {
    if (x >= 0 && x < 100 &&
        y >= 0 && y < 100) {
      return CACHED[x][y];
    } else {
      return new Point(x, y);
    }
  }
}
```

**Pedro:** For this pattern we need two things: precompute
most used points at a startup time, and use static factory
method instead of constructor to return cached object.  
**Eve:** Have you tested it?  
**Pedro:** Sure, the system works like a clock.  
**Eve:** Excellent, here is my version  

``` clojure
(defn make-point [x y]
  [x y {:some "Important Properties"}])

(def CACHE
  (let [cache-keys (for [i (range 100) j (range 100)] [i j])]
	  (zipmap cache-keys (map #(apply make-point %) cache-keys))))

(defn make-point-cached [x y]
  (let [result (get CACHE [x y])]
    (if result
	  result
	  (make-point x y))))
```

**Eve:** It creates a flat map with pair `[x, y]` as a key, instead of two-dimensional array.  
**Pedro:** Pretty the same.  
**Eve:** No, it is much flexible, you can't use two-dimensional array if you need to cache three points or non-integer values.  
**Pedro:** Oh, got it.  
**Eve:** Even better, in clojure you can just use `memoize` function to cache calls to factory function `make-point`  

``` clojure
(def make-point-memoize (memoize make-point))
```

**Eve:** Every call (except first one) with the same parameters
return cached value.  
**Pedro:** That's awesome!  
**Eve:** Of course, but remember **if your function has side-effects, memoization is bad idea**.  

### <div id="builder"/>Episode 13: Builder

> **Tuck Brass** complains that his old automatic
> coffee-making system is very slow in usage.
> Customers can't wait long enough and going away.

**Pedro:** Need to understand what's the problem exactly?  
**Eve:** I've researched, system is old, written in *COBOL* and built around expert system *question-answer*. They were very popular long time ago.  
**Pedro:** What do you mean by *"question-answer"*  
**Eve:** There is operator in front of terminal. System asks: *"Do yo want to add water?"*, operator answers *"Yes"*. Then system asks again: *"Do you want to add coffee"*, operator answers *"Yes"* and so forth.  
**Pedro:** It's a nightmare, I just want coffee with milk. Why they don't use predefined options: coffee with milk, coffee with sugar and so on.  
**Eve:** Because it is the raisin of the system: customer can make coffee with any set of ingridients *by themselves*  
**Pedro:** Okay, let's fix it with Builder pattern.  

``` java
public class Coffee {
  private String coffeeName; // required
  private double amountOfCoffee; // required
  private double water; // required
  private double milk; // optional
  private double sugar; // optional
  private double cinnamon; // optional

  private Coffee() { }

  public static class Builder {
    private String builderCoffeeName;
    private double builderAmountOfCoffee; // required
    private double builderWater; // required
    private double builderMilk; // optional
    private double builderSugar; // optional
    private double builderCinnamon; // optional

    public Builder() { }

    public Builder setCoffeeName(String name) {
      this.builderCoffeeName = name;
      return this;
    }

    public Builder setCoffee(double coffee) {
      this.builderAmountOfCoffee = coffee;
      return this;
    }

    public Builder setWater(double water) {
      this.builderWater = water;
      return this;
    }

    public Builder setMilk(double milk) {
      this.builderMilk = milk;
      return this;
    }

    public Builder setSugar(double sugar) {
      this.builderSugar = sugar;
      return this;
    }

    public Builder setCinnamon(double cinnamon) {
      this.builderCinnamon = cinnamon;
      return this;
    }

    public Coffee make() {
      Coffee c = new Coffee();
        c.coffeeName = builderCoffeeName;
        c.amountOfCoffee = builderAmountOfCoffee;
        c.water = builderWater;
        c.milk = builderMilk;
        c.sugar = builderSugar;
        c.cinnamon = builderCinnamon;

      // check required parameters and invariants
      if (c.coffeeName == null || c.coffeeName.equals("") ||
          c.amountOfCoffee <= 0 || c.water <= 0) {
        throw new IllegalArgumentException("Provide required parameters");
      }

      return c;
    }
  }
}
```

**Pedro:** As you see, you can't instantiate `Coffee` class easily, you need to set parameters with nested `Builder` class

``` java
Coffee c = new Coffee.Builder()
        .setCoffeeName("Royale Coffee")
        .setCoffee(15)
        .setWater(100)
        .setMilk(10)
        .setCinnamon(3)
        .make();
```

**Pedro:** Calling to method `make` checks all required parameters, and could validate and throw an exception if object is in inconsistent state.  
**Eve:** Awesome functionality, but why so verbose?  
**Pedro:** Beat it.  
**Eve:** A piece of cake, clojure supports **optional arguments**, everything what builder pattern is about.  

``` clojure
(defn make-coffee [name amount water
                   & {:keys [milk sugar cinnamon]
                      :or {milk 0 sugar 0 cinnamon 0}}]
  ;; definition goes here
  )

(make-coffee "Royale Coffee" 15 100
             :milk 10
             :cinnamon 3)
```

**Pedro:** Aha, you have three required parameters and three optionals, but required parameters still without names.  
**Eve:** What do you mean?  
**Pedro:** From the client call I see number `15` but I have no idea what it might be.  
**Eve:** Agreed. Then, let's make all parameters are named and add precondition for required, the same way you do with the builder.

```clojure
(defn make-coffee
  [& {:keys [name amount water milk sugar cinnamon]
      :or {name "" amount 0 water 0 milk 0 sugar 0 cinnamon 0}}]
  {:pre [(not (empty? name))
         (> amount 0)
         (> water 0)]}
  ;; definition goes here		 
  )

(make-coffee :name "Royale Coffee"
             :amount 15
             :water 100
             :milk 10
             :cinnamon 3)
```

**Eve:** As you see all parameters are named and all required params are checked in `:pre` constraint. If constraints are violated `AssertionError` is thrown.  
**Pedro:** Interesting, `:pre` is a part of a language?  
**Eve:** Sure, it's just a simple assertion. There is also `:post` constraint, with the similar effect.  
**Pedro:** Hm, okay. But as you know `Builder` pattern often used as a mutable datastucture, `StringBuilder` for example.  
**Eve:** It's not a part of clojure philosophy to use mutables, but if you *really* want, no problem.
Just create a new class with `deftype` and do not forget to use `volatile-mutable` on the properties you want to mutate.  
**Pedro:** Where is the code?  
**Eve:** Here is example of custom implementation of mutable `StringBuilder` in clojure. It has a lot of drawbacks and limitations but you've got the idea.

```clojure
;; interface
(defprotocol IStringBuilder
	(append [this s])
	(to-string [this]))

;; implementation
(deftype ClojureStringBuilder [charray ^:volatile-mutable last-pos]
	IStringBuilder
	(append [this s] 
	  (let [cs (char-array s)]
	    (doseq [i (range (count cs))]
	      (aset charray (+ last-pos i) (aget cs i))))
	  (set! last-pos (+ last-pos (count s))))
	(to-string [this] (apply str (take last-pos charray))))

;; clojure binding
(defn new-string-builder []
  (ClojureStringBuilder. (char-array 100) 0))

;; usage
(def sb (new-string-builder))
(append sb "Toby Wong")
(to-string sb) => "Toby Wong"
(append sb " ")
(append sb "Toby Chung") => "Toby Wang Toby Chung"
```

**Pedro:** Not as hard as I thought.

###<div id="facade"/> Episode 14: Facade

> Our new member **Eugenio Reinn Jr.** commited
> file with diff in 134 lines to processing servlet.
> But actual work is just to process request.
> All other code injection and imports.
> It MUST be one line commit.

**Pedro:** Who cares how many lines are committed?  
**Eve:** Someone cares.  
**Pedro:** Let's see where is the problem  

``` java
class OldServlet {
  @Autowired
  RequestExtractorService requestExtractorService;
  @Autowired
  RequestValidatorService requestValidatorService;
  @Autowired
  TransformerService transformerService;
  @Autowired
  ResponseBuilderService responseBuilderService;

  public Response service(Request request) {
    RequestRaw rawRequest = requestExtractorService.extract(request);
    RequestRaw validated = requestValidatorService.validate(rawRequest);
    RequestRaw transformed = transformerService.transform(validated);
    Response response = responseBuilderService.buildResponse(transformed);
    return response;
  }
}
```

**Eve:** Oh shi...  
**Pedro:** That's our internal API for developers, every time they need to process request, inject
4 services, include all imports, and write this code.  
**Eve:** Let's refactor it with...  
**Pedro:** ...Facade pattern. We resolve all dependencies to a **single point of access** and simplify API usage.  

``` java
public class FacadeService {
  @Autowired
  RequestExtractorService requestExtractorService;
  @Autowired
  RequestValidatorService requestValidatorService;
  @Autowired
  TransformerService transformerService;
  @Autowired
  ResponseBuilderService responseBuilderService;

  RequestRaw extractRequest(Request req) {
    return requestExtractorService.extract(req);
  } 
  
  RequestRaw validateRequest(RequestRaw raw) {
    return requestValidatorService.validate(raw);
  }
  
  RequestRaw transformRequest(RequestRaw raw) {
    return transformerService.transform(raw);
  }
  
  Response buildResponse(RequestRaw raw) {
    return responseBuilderService.buildResponse(raw);
  }
}
```

**Pedro:** Then if you need any service or set of services in the code
you just injecting facade to your code  

``` java
class NewServlet {
  @Autowired
  FacadeService facadeService;

  Response service(Request request) {
    RequestRaw rawRequest = facadeService.extractRequest(request);
    RequestRaw validated = facadeService.validateRequest(rawRequest);
    RequestRaw transformed = facadeService.transformRequest(validated);
    Response response = facadeService.buildResponse(transformed);
    return response;
  }
}
```

**Eve:** Wait, you've just moved all dependencies to one and everytime using this one, correct?  
**Pedro:** Yes, now everytime some functionality is needed, use `FacadeService`. Dependency is already there.  
**Eve:** But we did the same in [Mediator](#mediator) pattern?  
**Pedro:** Mediator is behavioral pattern. We resolved all dependency to Mediator and added *new behavior* to it.  
**Eve:** And facade?  
**Pedro:** Facade is structural, we don't add new functionality, we just *expose existing functionality* with facade.  
**Eve:** Got it. But seems that pattern very loud word for such little tweak.  
**Pedro:** Maybe.  
**Eve:** Here is clojure version using structure by namespaces  

``` clojure
(ns application.old-servlet
  (:require [application.request-extractor :as re])
  (:require [application.request-validator :as rv])
  (:require [application.transformer :as t])
  (:require [application.response-builder :as rb]))

(defn service [request]
  (-> request
      (re/extract)
      (rv/validate)
	  (t/transform)
	  (rb/build)))
```

**Eve:** Exposing all services via facade.  

``` clojure
(ns application.facade
  (:require [application.request-extractor :as re])
  (:require [application.request-validator :as rv])
  (:require [application.transformer :as t])
  (:require [application.response-builder :as rb]))

(defn request-extract [request]
  (re/extract request))

(defn request-validate [request]
  (rv/validate request))

(defn request-transform [request]
  (t/transform request))

(defn response-build [request]
  (rb/build request))
```

**Eve:** And use it.  

``` clojure
(ns application.old-servlet
  (:use [application.facade]))

(defn service [request]
  (-> request
      (request-extract)
      (request-validate)
	  (request-transform)
	  (request-build)))
```

**Pedro:** What the difference between `:use` and `:require`?  
**Eve:** They are almost similar, but with `:require` you expose functionality via namespace qualificator
`(namespace/function)` where with `:use` you can refer to it directly `(function)`  
**Pedro:** So, `:use` is better.  
**Eve:** No, be aware of `:use` because it can conflict with existing names in your namespace.  
**Pedro:** Oh, I see your point. And every time you call `(:use [application.facade])` in some namespace
all existing functionality from facade is available?  
**Eve:** Yes.  
**Pedro:** Pretty the same.  

###<div id="singleton"/> Episode 15: Singleton

> **Feverro O'Neal** complains that we have a lot of different styles for UI.  
> Force one per application UI configuration.

**Pedro:** But wait, there was requirement to save UI style per user.  
**Eve:** Probably it was changed.  
**Pedro:** Ok, then we should just save configuration to `Singleton` and use it from all the places.

``` java
public final class UIConfiguration {
  public static final UIConfiguration INSTANCE = new UIConfiguration("ui.config");

  private String backgroundStyle;
  private String fontStyle;
  /* other UI properties */

  private UIConfiguration(String configFile) {
    loadConfig(configFile);
  }

  private static void loadConfig(String file) {
    // process file and fill UI properties
	INSTANCE.backgroundStyle = "black";
    INSTANCE.fontStyle = "Arial";
  }

  public String getBackgroundStyle() {
    return backgroundStyle;
  }

  public String getFontStyle() {
    return fontStyle;
  }
}
```

**Pedro:** That way all configuration will be shared across the UIs.  
**Eve:** Yes, but...why so much code?  
**Pedro:** We guarantee that only one instance of `UIConfiguration` will exist.  
**Eve:** Let me ask you: what's the difference between singleton and global varaible.  
**Pedro:** What?  
**Eve:** ...the difference between singleton and global variable.  
**Pedro:** Java does not support global variables.  
**Eve:** But `UIConfiguration.INSTANCE` is global variable.  
**Pedro:** Well, sort of.  
**Eve:** Your code is just simple `def` in clojure.  

``` clojure
(def ui-config (load-config "ui.config"))

(defn load-config [config-file]
  ;; process config file and return map with configuratios
  {:bg-style "black" :font-style "Arial"})
```

**Pedro:** But, how do you change the style?  
**Eve:** The same way you will change it in your code.  
**Pedro:** Uhm... Ok, we need a simple tweak. Make `UIConfiguration.loadConfig` is public and
call it when the configuration changes.  
**Eve:** Then we make `ui-config` an atom and call `swap!` when configuration changes.  
**Pedro:** But atoms are useful only in concurrent environment.  
**Eve:** First, yes, they useful, but NOT only in concurrent environment.
Second, atom read is not as slow as you think. Third, it changes the state of
UI configuration *atomically*  
**Pedro:** It is redundant for such simple example.  
**Eve:** No, it is not. There is a posibility that UI configuration changes
and some renders read new `backgroundStyle`, but old `fontStyle`  
**Pedro:** Ok, use `synchronized` for `loadConfig`  
**Eve:** Then you must use `synchonized` on getters as well, it is slow.  
**Pedro:** There is still *Double-Checked Locking* idiom  
**Eve:** Double-checked locking is clever but [broken](http://www.javaworld.com/article/2074979/java-concurrency/double-checked-locking--clever--but-broken.html)  
**Pedro:** Ok, I give up, you won.

###<div id="chain"/> Episode 16: Chain Of Responsibility

> New York marketing organization **"A Profit NY"** opened request to
> filter profanity words from their public chat system.

**Pedro:** Fuck, they don't like the word "*fuck*"?  
**Eve:** It is profit organization, they lose money if someone use
profanity words in public chat.  
**Pedro:** Who defined profanity words list?  
**Eve:** [George Carlin](https://www.youtube.com/watch?v=5ssJtD08vCc)  

*Watching and laughing*

**Pedro:** Ok, so let's just add a filter to replace these rude words with the asterisks.  
**Eve:** Make sure your solution is extendable, other filters could be applied.  
**Pedro:** Chain of Responisibility seems like a good pattern candidate for that.
First of all we make some abstract filter.

``` java
public abstract class Filter {
  protected Filter nextFilter;

  abstract void process(String message);

  public void setNextFilter(Filter nextFilter) {
    this.nextFilter = nextFilter;
  }
}
```

**Pedro:** Then, provide implementation for each specific filter you want to apply

``` java
class LogFilter extends Filter {
  @Override
  void process(String message) {
    Logger.info(message);
    if (nextFilter != null) nextFilter.process(message);
  }
}

class ProfanityFilter extends Filter {
  @Override
  void process(String message) {
    String newMessage = message.replaceAll("fuck", "f*ck");
    if (nextFilter != null) nextFilter.process(newMessage);
  }
}

class RejectFilter extends Filter {
  @Override
  void process(String message) {
    System.out.println("RejectFilter");
    if (message.startsWith("[A PROFIT NY]")) {
      if (nextFilter != null) nextFilter.process(message);
    } else {
      // reject message - do not propagate processing
    }
  }
}

class StatisticsFilter extends Filter {
  @Override
  void process(String message) {
    Statistics.addUsedChars(message.length());
    if (nextFilter != null) nextFilter.process(message);
  }
}
```

**Pedro:** And finally build a chain of filters which defines an order how message will be processed.

``` java
Filter rejectFilter = new RejectFilter();
Filter logFilter = new LogFilter();
Filter profanityFilter = new ProfanityFilter();
Filter statsFilter = new StatisticsFilter();

rejectFilter.setNextFilter(logFilter);
logFilter.setNextFilter(profanityFilter);
profanityFilter.setNextFilter(statsFilter);

String message = "[A PROFIT NY] What the fuck?";
rejectFilter.process(message);
```

**Eve:** Ok, now clojure turn. Just define each filter as a function.

``` clojure
;; define filters

(defn log-filter [message]
  (logger/log message)
  message)

(defn stats-filter [message]
  (stats/add-used-chars (count message))
  message)

(defn profanity-filter [message]
  (clojure.string/replace message "fuck" "f*ck"))

(defn reject-filter [message]
  (if (.startsWith message "[A Profit NY]")
    message))
```

**Eve:** And use `some->` macro to chain filters

``` clojure
(defn chain [message]
  (some-> message
          reject-filter
          log-filter
          stats-filter
          profanity-filter))
```

**Eve:** You see how much it is easier, you don't need every-time call
`if (nextFilter != null) nextFilter.process()`, because it's natural.
The next filter defined at the `some->` level naturally, instead of calling manuall `setNext`.  
**Pedro:** That's definitely better for composability, but why did you use `some->` instead of `->`?  
**Eve:** Just for `reject-filter`. It could stop further processing, so `some->` returns `nil` as soon
as `nil` encountered as a filter  
**Pedro:** Could you explain more?  
**Eve:** Look at the usage  

``` clojure
(chain "fuck") => nil
(chain "[A Profit NY] fuck") => "f*ck"
```

**Pedro:** Understood.  
**Eve:** *Chain of Responsibility* just an approach to **function composition**  

###<div id="composite"/> Episode 17: Composite

> Actress **Bella Hock** don't see user avatars in our social network on her computer.
> 
> "Everything is black. Is it black holes?"

**Pedro:** It is black squares.  
**Eve:** Hmm, the same problem on our side.  
**Pedro:** Seems, that latest feature broke user avatars.  
**Eve:** Strange, because avatars rendered the same way as other elements, but they are visible.  
**Pedro:** Are you sure it rendered the same way?  
**Eve:** Well...no  

*Digging code*  

**Pedro:** What the hell is going on here?  
**Eve:** Someone copypasted code, but forgot to reflect changes in avatars.  
**Pedro:** Let's blame him, `git-blame`  
**Eve:** *Blame* is good, but we need to fix the problem  
**Pedro:** It's simple just additional line here.  
**Eve:** I mean, really solve the problem. Why do we need two similar snippets of code to
process the same blocks?  
**Pedro:** Correct, I think we could use Composite pattern to handle rendering of the whole page.
Our smallest element to render is block.  


``` java
public interface Block {
  void addBlock(Block b);
  List<Block> getChildren();
  
  void render();
}
```

**Pedro:** Obviously blocks may contain other blocks, that's the point of Composite pattern.
We may create some specific blocks.  

``` java
public class Page implements Block { }
public class Header implements Block { }
public class Body implements Block { }
public class HeaderTitle implements Block { }
public class UserAvatar implements Block { }
```

**Pedro:** And treat every specific element as a `Block`  

``` java
Block page = new Page();
Block header = new Header();
Block body = new Body();
Block title = new HeaderTitle();
Block avatar = new UserAvatar();

page.addBlock(header);
page.addBlock(body);
header.addBlock(title);
header.addBlock(avatar);

page.render();
```

**Pedro:** This is a structural pattern, a good way to *compose* objects. That's why it is called *composite*  
**Eve:** Hey, **composite is a simple tree** structure.  
**Pedro:** Yes.  
**Eve:** There is pattern for every datastructure?  
**Pedro:** No, just for list and tree.  
**Eve:** Actually, tree can be represented as a list  
**Pedro:** How?  
**Eve:** First element of a list is a node value, next elements are children, each of them is...  
**Pedro:** I understand.  
**Eve:** To be specific, here is the tree  

```
        A
     /  |  \
    B   C   D
    |   |  / \
	E   H J   K
   / \       /|\
  F   G     L M N
```

**Eve:** And here is the list represents this tree  

``` clojure
(def tree
  '(A (B (E (F) (G))) (C (H)) (D (J) (K (L) (M) (N)))))
```

**Pedro:** It's a plenty of parentheses!  
**Eve:** They define structure, you know  
**Pedro:** But it's hard to understand  
**Eve:** It's easy for machine, there is awesome `tree-seq` function to process the tree.  

``` clojure
(map first (tree-seq next rest tree)) => (A B E F G C H D J K L M N)
```

**Eve:** If you need more advanced traversals, use [clojure.walk](https://clojure.github.io/clojure/clojure.walk-api.html)  
**Pedro:** I don't know, everything seems just a bit harder.  
**Eve:** No, you define the whole tree with one datastructure and use one function to operate on it.  
**Pedro:** What this function will do?  
**Eve:** It traverses the tree and applies to every node, so in our case it can render each component.  
**Pedro:** I don't know, maybe I am too young for the trees, let's move forward.  

###<div id="factory_method"/> Episode 18. Factory Method

> **Sir Dry Bang** suggest to create new levels for their popular
> game. More levels - more money.

**Pedro:** How can we create new levels?  
**Eve:** Just change assets and add new blocks: paper, wood, iron...  
**Pedro:** It's too silly, isn't it?  
**Eve:** The whole game is too silly. If user pay for color hats for their characters, then they will pay for wooden blocks as well.  
**Pedro:** I think it's a crap, but anyway, let's make `MazeBuilder` generic and add specific builder
for each type of the block. It's a Factory Method pattern.  

``` java
class Maze { }
class WoodMaze extends Maze { }
class IronMaze extends Maze { }

interface MazeBuilder {
  Maze build();
}

class WoodMazeBuilder {
  @Override
  Maze build() {
    return new WoodMaze();
  }
}

class IronMazeBuilder {
  @Override
  Maze build() {
    return new IronMaze();
  }
}
```

**Eve:** Isn't it obvious that IronMazeBuilder will return IronMazes?  
**Pedro:** Not for program. But see, to make a maze from another blocks we just change implementation responsible for block creation.  

``` java
MazeBuilder builder = new WoodMazeBuilder();
Maze maze = builder.build();
```

**Eve:** I've seen something similar before.  
**Pedro:** What exactly?  
**Eve:** For me it seems like a strategy or state pattern.  
**Pedro:** No way! Strategy is about performing specific operations and factory is for creating specific object.  
**Eve:** But create is an operation as well.  

``` clojure
(defn maze-builder [maze-fn])

(defn make-wood-maze [])
(defn make-iron-maze [])

(def wood-maze-builder (partial maze-builder make-wood-maze))
(def iron-maze-builder (partial maze-builder make-iron-maze))
```

**Pedro:** Hm, seems similar.  
**Eve:** Think about it.  
**Pedro:** Any usage examples?  
**Eve:** No, everything is obvious here, just re-read [Strategy](#strategy), [State](#state) or [TemplateMethod](#template_method)
episodes.

###<div id="abstract_factory"/> Episode 19: Abstract Factory

> Users are not buying new levels in the game.
> **Saimank Gerr** build a complains cloud and the most popular
> negative feedback words are: "ugly", "crap" and "shit".
>
> Improve levels-building system.

**Pedro:** I said this is crap.  
**Eve:** Sure, snow background with wooden walls, space invaders with wooden walls every setting with wooden walls.  
**Pedro:** Then we must separate game worlds and build a set of specific objects for particular world.  
**Eve:** Explain.  
**Pedro:** Instead of using Factory Method for building specific blocks, we use Abstract Factory
to build a set of related objects, to make a level look less crappy.  
**Eve:** Example would be good.  
**Pedro:** Code is my example. First we define *abstract* behaviour of level factory  

``` java
public interface LevelFactory {
  Wall buildWall();
  Back buildBack();
  Enemy buildEnemy();
}
```

**Pedro:** Then we have a hierarchy of objects that level consists of  

``` java
class Wall {}
class PlasmaWall extends Wall {}
class StoneWall extends Wall {}

class Back {}
class StarsBack extends Back {}
class EarthBack extends Back {}

class Enemy {}
class UFOSoldier extends Enemy {}
class WormScout extends Enemy {}
```

**Pedro:** See? We have a specific object for each level, let's create factory for them.  

``` java
class SpaceLevelFactory implements LevelFactory {
  @Override
  public Wall buildWall() {
    return new PlasmaWall();
  }

  @Override
  public Back buildBack() {
    return new StarsBack();
  }

  @Override
  public Enemy buildEnemy() {
    return new UFOSoldier();
  }
}

class UndergroundLevelFactory implements LevelFactory {
  @Override
  public Wall buildWall() {
    return new StoneWall();
  }

  @Override
  public Back buildBack() {
    return new EarthBack();
  }

  @Override
  public Enemy buildEnemy() {
    return new WormScout();
  }
}
```

**Pedro:** Each implementation of level factory creates related objects for level.
Levels will look less crappy for sure.  
**Eve:** Let me understand. I really can't spot the difference.  
**Pedro:** Factory Method *defers object creation to a subclasses*,
Abstract Factory do the same but for a *set of related object*.  
**Eve:** Aha, that means I need to pass **set of related functions** to abstract builder  

``` clojure
(defn level-factory [wall-fn back-fn enemy-fn])

(defn make-stone-wall [])
(defn make-plasma-wall [])

(defn make-earth-back [])
(defn make-stars-back [])

(defn make-worm-scout [])
(defn make-ufo-soldier [])

(def underground-level-factory
  (partial level-factory
           make-stone-wall
           make-earth-back
           make-worm-scout))

(def space-level-factory
  (partial level-factory
           make-plasma-wall
           make-stars-back
           make-ufo-soldier))
```

**Pedro:** I knew.  
**Eve:** Everything is fair. Your lovely "set of related Xs", where X is a function  
**Pedro:** Yes, clarify, what `partial` is.  
**Eve:** Provide some parameters for function. So, `underground-level-factory` knows how to construct walls, backs and enemies. Everything other *inherited* from abstract `level-factory` function.  
**Pedro:** Handy.  

###<div id="adapter"/> Episode 20: Adapter

> **Deam Evil** conducts a medieval tournament for knights.
> The prize is $100.000
>
> I'll pay you the half if you break the system and allow
> my armed commando to take part in competition.

**Pedro:** Finally we've got interesting work.  
**Eve:** Funny to see the competition. Especially, M16 vs Iron Sword part.  
**Pedro:** Knights have a good armor.  
**Eve:** F1 grenade *does not care* about armor.  
**Pedro:** Nevermind, we do the work, we get the money.  
**Eve:** Fifty grands - nice compensation  
**Pedro:** Yes, look at this, I've stolen sources of the competition system,
though it is not possible to modify their sources, we can find some vulneravility.  
**Eve:** Here it is  

``` java
public interface Tournament {
  void accept(Knight knight);
}
```

**Pedro:** Aha! System validates only incoming types via `Knight` interface.
All we need to do is *to adapt* commando to be a knight. Let's see how knight look like  

``` java
interface Knight {
  void attackWithSword();
  void attackWithBow();
  void blockWithShield();
}

class Galahad implements Knight {
  @Override
  public void blockWithShield() {
    winkToQueen();
    take(shield);
    block();
  }

  @Override
  public void attackWithBow() {
    winkToQueen();
    take(bow);
    attack();
  }

  @Override
  public void attackWithSword() {
    winkToQueen();
    take(sword);
    attack();
  }
}
```

**Pedro:** To accept the commando let's take an old implementation  

``` java
class Commando {
    void throwGrenade(String grenade) { }
    shot(String rifleType) { }
}
```

**Pedro:** And adapt it.  

``` java
class Commando implements Knight {
  @Override
  public void blockWithShield() {
    // commando don't block
  }

  @Override
  public void attackWithBow() {
    throwGrenade("F1");
  }

  @Override
  public void attackWithSword() {
    shotWithRifle("M16");
  }
}
```

**Pedro:** That's it.  
**Eve:** It's simpler in clojure.  
**Pedro:** Really?  
**Eve:** We don't love types so their validation won't work at all  
**Pedro:** So, how do you replace knight with commando?  
**Eve:** Basically, what knight is? It's a map, consists of data and behaviour  

``` clojure
{:name "Lancelot"
 :speed 1.0
 :attack-bow-fn attack-with-bow
 :attack-sword-fn attack-with-sword
 :block-fn block-with-shield}
```

**Eve:** To adapt commando, just pass his functions instead original ones  

``` clojure
{:name "Commando"
 :speed 5.0
 :attack-bow-fn (partial throw-grenade "F1")
 :attack-sword-fn (partial shot "M16")
 :block-fn nil}
```

**Pedro:** How did we share money?  
**Eve:** 50/50  
**Pedro:** I wrote more code, I want 70  
**Eve:** Ok, 70/70  
**Pedro:** Deal.  

###<div id="decorator"/> Episode 21: Decorator

> **Podrea Vesper** caught us on cheating for the tournament.
> We have a choice: to be busted by police or to help
> his super knight to take part in competition

**Pedro:** I don't wanna go to prison.  
**Eve:** Me either.  
**Pedro:** Let's cheat for him one more time.  
**Eve:** This is the same solution, isn't it?  
**Pedro** Similar, but not the same.
Commando was a soldier and they are not allowed on the tournament. We *adapted* it. But knight is allowed for tournament we don't need to adapt it, we *must* add functionality to existing object.  
**Eve:** Inheritance or composition?  
**Pedro:** Composition, the main goal of decorator to change behaviour at a runtime  
**Eve:** So how we deal with this super knight?  
**Pedro:** They plan to use `Galahad` knight and *decorate* it with *more HP* and *Power Armor*  
**Eve:** Heh, funny that cops are playing Fallout  
**Pedro:** Yeah, let's make knight an abstract class  

``` java
public class Knight {
    protected int hp;
    private Knight decorated;

    public Knight() { }

    public Knight(Knight decorated) {
        this.decorated = decorated;
    }

    public void attackWithSword() {
        if (decorated != null) decorated.attackWithSword();
    }

    public void attackWithBow() {
        if (decorated != null) decorated.attackWithBow();
    }

    public void blockWithShield() {
        if (decorated != null) decorated.blockWithShield();
    }
}
```

**Eve:** So what we improved there?  
**Pedro:** First af all we make class `Knight` instead of interface to have access to hit points. Then we provide two diferent constructors, default for standard behaviour, and decorated, which delegates call to decorated object.  
**Eve:** Is it fair to use abstract class instead of interface?  
**Pedro:** No, but we avoid two classes with similar behaviour and fill each decorated object with default implementation, instead of forcing to implement each of the methods.  
**Eve:** Ok, what's about Power Armor?  
**Pedro:** Easy as well  

``` java
public class KnightWithPowerArmor extends Knight {
    public KnightWithPowerArmor(Knight decorated) {
        super(decorated);
    }

    @Override
    public void blockWithShield() {
        super.blockWithShield();
        Armor armor = new PowerArmor();
        armor.block();
    }
}

public class KnightWithAdditionalHP extends Knight {
    public KnightWithAdditionalHP(Knight decorated) {
        super(decorated);
        this.hp += 50;
    }
}
```

**Pedro:** Two decorators that fulfils FBI requirements, and we able
to create super knight, with behaviour like Galahad, but super armor and 50 more hit points.

``` java
Knight superKnight =
     new KnightWithAdditionalHP(
     new KnightWithPowerArmor(
	 new Galahad()));
```

**Eve:** Nice trick!  
**Pedro:** You are welcome to show the similar behaviour in clojure  
**Eve:** Here it is  

``` clojure
(def galahad {:name "Galahad"
              :speed 1.0
              :hp 100
              :attack-bow-fn attack-with-bow
              :attack-sword-fn attack-with-sword
              :block-fn block-with-shield})

(defn make-knight-with-more-hp [knight]
  (update-in knight [:hp] + 50))

(defn make-knight-with-power-armor [knight]
  (update-in knight [:block-fn]
             (fn [block-fn]
               (fn []
                 (block-fn)
                 (block-with-power-armor)))))

;; create the knight
(def superknight (-> galahad
                     make-knight-with-power-armor
                     make-knight-with-more-hp)
```

**Pedro:** The same functionality.  
**Eve:** Yes, just pay attention to power armor decorator.  



###<div id="proxy"/> Episode 22: Proxy

> **Deren Bart** manages the system for making mixed drinks.
> It is rigid system, because after making a drink *Bart* must manually subtract used ingredients
> from the bar. Do this automatically.

**Pedro:** Can we get access to his codebase?  
**Eve:** No, but he sent some APIs.  

```java
interface IBar {
    void makeDrink(Drink drink);
}

interface Drink {
    List<Ingredient> getIngredients();
}
    
interface Ingredient {
    String getName();
    double getAmount();
}
```

**Pedro:** Bart doesn't want us to modify sources, instead we need to provide some additional
implementation for `IBar` interface with autosubtracting used ingredients.  
**Eve:** And what are we responsible for?  
**Pedro:** Implementing _Proxy Pattern_, I've read about it some times ago.  
**Eve:** I'm all listening.  
**Pedro:** Basically we delegate all existing functionality to standard `IBar` implementation
and provide new functionality inside `ProxiedBar`

```java
class ProxiedBar implements IBar {
    BarDatabase bar;
    IBar standardBar;

    public void makeDrink(Drink drink) {
       standardBar.makeDrink(drink);
       for (Ingredient i : drink.getIngredients()) {
           bar.subtract(i);
       }
    }
}
```

**Pedro:** They need to replace `StandardBar` implementation with our `ProxiedBar`.  
**Eve:** Seems super easy.  
**Pedro:** Yes, additional plus that we don't break existing functionality.  
**Eve:** Are you sure? We didn't run regression tests.  
**Pedro:** Everything we do is delegating functionality to already tested `StandardBar`  
**Eve:** But you also substracts used ingredients from `BarDatabase`  
**Pedro:** We assume, they are _decoupled_.  
**Eve:** Oh...  
**Pedro:** Does clojure have some alternative?  
**Eve:** Well, I don't know. What I see here you are using function composition.  
**Pedro:** Explain.  
**Eve:** `IBar` implementation is a set of functions, and another `IBar` is another set of functions.
Everything you talking about additional implementation could be covered by function composition. It's like
`make-drink` and after that `subtract-ingredients` from bar.  
**Pedro:** Maybe code is more clear?  
**Eve:** Yes, but I don't think something special here  

```clojure
;; interface
(defprotocol IBar
  (make-drink [this drink]))

;; Bart's implementation
(deftype StandardBar []
  IBar
  (make-drink [this drink]
    (println "Making drink " drink)
    :ok))

;; our implementation
(deftype ProxiedBar [db ibar]
  IBar
  (make-drink [this drink]
    (make-drink ibar drink)
    (subtract-ingredients db drink)))

;; this how it was before
(make-drink (StandardBar.)
	{:name "Manhattan"
	 :ingredients [["Bourbon" 75] ["Sweet Vermouth" 25] ["Angostura" 5]]})

;; this how it becomes now
(make-drink (ProxiedBar. {:db 1} (StandardBar.))
	{:name "Manhattan"
     :ingredients [["Bourbon" 75] ["Sweet Vermouth" 25] ["Angostura" 5]]})
```

**Eve:** We could leverage protocol and types to group set of functions as a single object.  
**Pedro:** Looks like clojure has object-oriented capabilities as well.  
**Eve:** Correct, moreover it has a `reify` function, which allow you to create proxies in a runtime  
**Pedro:** Like class in a runtime?  
**Eve:** Sort of.

```clojure
(reify IBar
  (make-drink [this drink]
    ;; implementation goes here
  ))
```

**Pedro:** Looks handy.  
**Eve:** Yes, but I still don't understand how it differs from Decorator.  
**Pedro:** They are completely different.  
**Eve:** Decorator adds functionality to the same interface, and so does Proxy.  
**Pedro:** Well, but Proxy is...  
**Eve:** Even more, Adapter is not very different as well.  
**Pedro:** It uses another interface.  
**Eve:** But from implementation perspective all these pattern are the same,
wrap something and delegate calls to wrapper. *"Wrapper"* could be a good name for these patterns.  

###<div id="bridge"/> Episode 23: Bridge

> Girls from HR agency "Hurece's Sour Man" trying to identify candidates
> to their open job positions. The problem is jobs often created by customers, but requirements to the jobs
> are developed by HR. Provide them with a flexible way of collaborating.

**Eve:** I don't understand the problem.  
**Pedro:** I have a bit of background. They have a very strange system which defines job requirements as an interface.

```java
interface JobRequirement {
    boolean accept(Candidate c);
}
```

**Pedro:** Every specific requirement implemented as a new subclass of this.

```java
class JavaRequirement implements JobRequirement {
    public boolean accept(Candidate c) {
        return c.hasSkill("Java");
    }
}

class Experience10YearsRequirement implements JobRequirement {
    public boolean accept(Candidate c) {
        return c.getExperience() >= 10;
    }
}
```

**Eve:** I've got an idea.  
**Pedro:** Take into account, this requirement hierarchy is designed by HR Department.  
**Eve:** Ok.  
**Pedro:** And they have a `Job` hierarchy, where each specific job is subclass as well.  
**Eve:** Why do they need a class for each job? It should be an object.  
**Pedro:** The system was designed when classes were more popular than objects, so live with this.  
**Eve:** Class were popular than objects?!  
**Pedro:** Yes, listen and don't interrupt me. Jobs with requirements is completely separate hierarchy,
and it is developed by customers. We introduce pattern `Bridge` to separate these two hierarchies and allow
them live independently.  

```java
abstract class Job {
    protected List<? extends JobRequirement> requirements;

    public Job(List<? extends JobRequirement> requirements) {
        this.requirements = requirements;
    }

	protected boolean accept(Candidate c) {
        for (JobRequirement j : requirements) {
            if (!j.accept(c)) {
                return false;
			}
        }
        return true;
    }
}

class CognitectClojureDeveloper extends Job {
    public CognitectClojureDeveloper() {
        super(Arrays.asList(
                  new ClojureJobRequirement(),
                  new Experience10YearsRequirement()
        ));
    }
}
```

**Eve:** So where is the bridge?  
**Pedro:** `JobRequirement`, `JavaRequirement`, `ExperienceRequirement` is one hierarchy, yes?  
**Eve:** Yes.  
**Pedro:** `Job`, `CongnitectClojureDeveloperJob`, `OracleJavaDeveloperJob` is another hierarchy.  
**Eve:** Oh, now I see. A link from Job to JobRequirement is a bridge.  
**Pedro:** Exactly! This is how HR can use this system to find candidate matches.  

```java
Candidate joshuaBloch = new Candidate();
(new CognitectClojureDeveloper()).accept(joshuaBloch);
(new OracleSeniorJavaDeveloper()).accept(joshuaBloch);
```

**Pedro:** Here is the point. Customers use `Job` as abstraction and `JobRequirement` as an implementation.
They just create a job with descriptions or so, and HR is responsible to convert these descriptions into specific set
of `JobRequirement` objects.  
**Eve:** Got it.  
**Pedro:** So as far as I understand, clojure could mimic this pattern by using `defprotocol` and `defrecord`?  
**Eve:** Yes, but I want to revisit the problem.  
**Pedro:** What's wrong?  
**Eve:** Here is we have a static flow: customer create job positions, human resources convert job position to a set of requirements
and run a script against their candidates database to find a matches.  
**Pedro:** Correct.  
**Eve:** So there is already dependency, HR can't do anything without open job positions.  
**Pedro:** Well, yes. But they can develop a set of structured requirements without knowing what open positions might be.  
**Eve:** For what purpose?  
**Pedro:** Later this can be reused by Job creators, so HR avoid doing the same work twice.  
**Eve:** Okay, got it, but this problem is artificial. Basically what we need is a way to colaborate between abstraction and implementation.  
**Pedro:** Maybe, but I want to see your clojure way solution for that specific problem using bridge pattern.  
**Eve:** Easy. Let's use adhoc hierarchies.  
**Pedro:** For abstractions?  
**Eve:** Yes, jobs hierarchy is _abstraction_, and people need just to enhance hierarchy.  


```clojure
;; abstraction

(derive ::clojure-job ::job)
(derive ::java-job ::job)
(derive ::senior-clojure-job ::clojure-job)
(derive ::senior-java-job    ::java-job)
```

**Eve:** HR department are like _developers_, they provide implementation for this abstraction.  

```clojure
;; implementation
(defmulti accept :job)

(defmethod accept :java [candidate]
  (and (some #{:java} (:skills candidate))
       (> (:experience candidate) 1)))
```

**Eve:** Later, when new jobs are created, but requirements are not yet developed and no `accept` method implementation for this type of job, we fallback using adhoc hierarchy.  
**Pedro:** Hm?  
**Eve:** Assume someone created a new `::senior-java` as a child of `::java` job.  
**Pedro:** Oh, and if HR not provided `accept` implementation for dispatch value `::senior-java`, method with dispatch value `::java` will be called, yes?  
**Eve:** You learning so fast.  
**Pedro:** But is it real bridge pattern?  
**Eve:** There is no _bridge_ here, but abstraction and implementation can live independently.

# The End.


### <div id="cheatsheet"/>Cheatsheet (instead of conclusion)

It is very confusing to understand patterns, which often presented in object-oriented way
with bunch of UML diagrams, fancy nouns and exist to solve language-specific problems,
so here is a revisited mini cheatsheet, which helps you to understand patterns by analogy.

- **Command** - _function_
- **Strategy** - _function, which accepts function_
- **State** - _strategy, depends on state_
- **Visitor** - _multiple dispatch_
- **Template Method** - _strategy with defaults_
- **Iterator** - _sequence_
- **Memento** - _save and restore_
- **Prototype** - _immutability_
- **Mediator** - _reduce coupling_
- **Observer** - _function, which calls after another function_
- **Interpreter** - _set of functions to process a tree_
- **Flyweight** - _cache_
- **Builder** - _optional arguments_
- **Facade** - _single point of access_
- **Singleton** - _global variable_
- **Chain of Responsibility** - _function composition_
- **Composite** - _tree_
- **Factory Method** - _strategy for creating objects_
- **Abstract Factory** - _strategy for creating set of related objects_
- **Adapter** - _wrapper, same functionality, different type_ 
- **Decorator** - _wrapper, same type, new functionality_
- **Proxy** - _wrapper, function composition_
- **Bridge** - _separate abstraction and implementation_

### Cast

> A long time ago in a galaxy far, far away...

With the lack of imagination all characters
and names are just anagrams.

**Pedro Veel** - Developer  
**Eve Dopler** - Developer  
**Serpent Hill & R.E.E.** - Enterprise Hell  
**Sven Tori** - Investor  
**Karmen Git** - Marketing  
**Natanius S. Selbys** - Business Analyst  
**Mech Dominore Fight Saga** - Heroes of Might and Magic  
**Kent Podiololis** - I don't like loops  
**Chad Bogue** - Douchebag  
**Dex Ringeus** - UX Designer  
**Veerco Wierde** - Code Review  
**Dartee Hebl** - Heartbleed  
**Bertie Prayc** - Cyber Pirate  
**Cristopher, Matton & Pharts** - Important Charts & Reports  
**Tuck Brass** - Starbucks  
**Eugenio Reinn Jr.** - Junior Engineer  
**Feverro O'Neal** - Forever Alone  
**A Profit NY** - Profanity  
**Bella Hock** - Black Hole  
**Sir Dry Bang** - Angry Birds  
**Saimank Gerr** - Risk Manager  
**Deam Evil** - Medieval  
**Podrea Vesper** - Eavesdropper  
**Deren Bart** - Bartender  
**Hurece's Sour Man** - Human Resources  

**P.S.** I've started to write this article _more than_ 2 years ago. Time has passed, things have changed and even Java 8 was released.
