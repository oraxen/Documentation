---
description: also known as "How to superpower Oraxen?"
---

# Create your own Mechanic

## How does the Mechanic system work?

### What's a mechanic?

A mechanic is a custom item property. There can be different variations of the same property on different items. For example, the durability property allows you to define a custom durability for an item, but not all items that have a durability mechanic have the same durability.

{% hint style="info" %}
The durability mechanic works by storing the item's durability in its metadatas and uses the vanilla durability bar to display its custom durability and not its vanilla durability
{% endhint %}

### So if a Mechanic is different for every Item, should I rewrite a different Durability everytime?

No, it would be far too long. Instead, Oraxen offers a MechanicFactory system. Basically you will have to create a Mechanics class that is configurable, and a MechanicsFactory class that will configure all the different versions of your Mechanic class. It will also manage the common code of all these versions.

### Okay, but I want to modify the items which implement my Mechanic to store data in it, is it possible?

Of course, for this purpose Oraxen allows you to associate a list of ItemModifier to each mechanic. An ItemModifier is `Function<ItemBuilder, ItemBuilder>` which is basically a small piece of code that contains changes to be made to an item when it is generated by the server via its configuration. For example for durability mechanics I use an itemModifier that stores the durability chosen by the user in the item's metadatas.

```java
item -> item.setCustomTag(NAMESPACED_KEY,
                        PersistentDataType.INTEGER, section.getInt("value"))
```

## Let's create our first mechanic

{% hint style="info" %}
For this tutorial I will take as an example the durability mechanic \(because it is very simple to understand\) but you can follow this tutorial to create the one you want.
{% endhint %}

### First step: create our mechanic class

Start by creating a class inheriting from Mechanic, if you use [intelliJ](https://www.jetbrains.com/idea/) you should get something like this:

```java
class DurabilityMechanic extends Mechanic {

    public DurabilityMechanic(MechanicFactory mechanicFactory, 
                    ConfigurationSection section,
                    Function<ItemBuilder, ItemBuilder>... modifiers) {
        super(mechanicFactory, section, modifiers);
    }

}
```

The Mechanic constructor takes three arguments:

```text
- An instance of the Factory which created the Mechanic
- The section used to configure the Mechanic
- The item modifier(s)
```

I want each variation of my mechanics to have a different durability, so I will read the configuration of the mechanics and store the value of the field value.

#### How the Mechanic configuration section will look like:

```java
class DurabilityMechanic extends Mechanic {

    private int itemDurability;

    public DurabilityMechanic(MechanicFactory mechanicFactory, 
                              ConfigurationSection section) {
        /* We give:
        - an instance of the Factory which created the mechanic
        - the section used to configure the mechanic
        - the item modifier(s)
         */
        super(mechanicFactory, section, item ->
                item.setCustomTag(NAMESPACED_KEY,
                        PersistentDataType.INTEGER, section.getInt("value")));
        this.itemDurability = section.getInt("value");
    }

    public int getItemMaxDurability() {
        return itemDurability;
    }
}
```

So now we have a DurabilityMechanic class that is able to adapt to any item and that will call our DurabilityModifier class to tell to Oraxen which modifications to do before creating them \(here we just add a data in the item which contains the wanted new durability\).

### Second step: create our mechanic factory class

As before, use your ide features to automatically create a class that extends MechanicFactory:

```java
class DurabilityMechanicFactory extends MechanicFactory {

    public DurabilityMechanicFactory(ConfigurationSection section) {
        super(section);
    }

    @Override
    public Mechanic parse(ConfigurationSection itemMechanicConfiguration) {
        return null;
    }
}
```

We rewrite the parse method to create a new Mechanic \(via the DurabilityMechanic class created previously\). We also want to tell to Oraxen that this mechanic is successfully implemented and can be loaded using `addToImplemented` method. So our class now looks like that:

```java
public class DurabilityMechanicFactory extends MechanicFactory {

    public DurabilityMechanicFactory(ConfigurationSection section) {
        super(section);
    }

    @Override
    public Mechanic parse(ConfigurationSection itemMechanicConfiguration) {
        Mechanic mechanic = new DurabilityMechanic(this, itemMechanicConfiguration);
        addToImplemented(mechanic);
        return mechanic;
    }

}
```

### Third step: add our features \(events\)

In my case I need to use only one event to play with durability and I will create a DurabilityMechanicsManager class that implements Listener to have a clean and tidy code but I could have done it directly in DurabilityMechanicFactory. I tell to Bukkit which class manages the events when the factory is built:

```java
public class DurabilityMechanicFactory extends MechanicFactory {

    public DurabilityMechanicFactory(ConfigurationSection section) {
        super(section);
        MechanicsManager.registerListeners(OraxenPlugin.get(),
                new DurabilityMechanicsManager(this));
    }

    @Override
    public Mechanic parse(ConfigurationSection itemMechanicConfiguration) {
        Mechanic mechanic = new DurabilityMechanic(this, itemMechanicConfiguration);
        addToImplemented(mechanic);
        return mechanic;
    }

}
```

In order to calculate the durability to display on the item according to the real durability managed by the plugin I use some simple maths:

$$
bukkitDamage = bukkitMaxDurability - \frac{realDurability*bukkitMaxDurability}{realMaxDurability}
$$

So this is my DurabilityMechanicsManager class:

```java
class DurabilityMechanicsManager implements Listener {

    private DurabilityMechanicFactory factory;

    public DurabilityMechanicsManager(DurabilityMechanicFactory factory) {
        this.factory = factory;
    }

    @EventHandler(priority = EventPriority.HIGH, ignoreCancelled = true)
    private void onItemDamaged(PlayerItemDamageEvent event) {
        ItemStack item = event.getItem();
        String itemID = OraxenItems.getIdByItem(item);
        if (factory.isNotImplementedIn(itemID))
            return;

        DurabilityMechanic durabilityMechanic = 
                (DurabilityMechanic) factory.getMechanic(itemID);

        ItemMeta itemMeta = item.getItemMeta();
        PersistentDataContainer persistentDataContainer = 
                itemMeta.getPersistentDataContainer();
        int realDurabilityLeft = persistentDataContainer
                .get(DurabilityMechanic.NAMESPACED_KEY, PersistentDataType.INTEGER) 
                        - event.getDamage();

        if (realDurabilityLeft > 0) {
            double realMaxDurability = 
                    //because int rounded values suck
                    durabilityMechanic.getItemMaxDurability();
            persistentDataContainer.set(DurabilityMechanic.NAMESPACED_KEY,
                    PersistentDataType.INTEGER, realDurabilityLeft);
            ((Damageable) itemMeta).setDamage((int) (item.getType()
                    .getMaxDurability() - realDurabilityLeft 
                    / realMaxDurability * item.getType().getMaxDurability()));
            item.setItemMeta(itemMeta);
        } else {
            item.setAmount(0);
        }

    }

}
```

### Last step: register our mechanic

Just call this line when you load your plugin \(e.g. in your onEnable method\):

```java
MechanicsManager.registerMechanicFactory("durability", 
                                DurabilityMechanicFactory.class);
```

## Conclusion

To properly create a new mechanic it is recommended to separate its code into three parts:

* A factory which extends [MechanicFactory](https://github.com/Th0rgal/Oraxen/blob/master/src/main/java/io/th0rgal/oraxen/items/mechanics/MechanicFactory.java)
* A mechanic which extends [Mechanic](https://github.com/Th0rgal/Oraxen/blob/master/src/main/java/io/th0rgal/oraxen/items/mechanics/Mechanic.java)
* Your own features in a &lt;YourMechanicName&gt;MechanicsManager class \(optional too\)

{% hint style="info" %}
While it is possible to modify items using your mechanic thanks to an ItemModifier:

```java
item -> item.setCustomTag(NAMESPACED_KEY,
                        PersistentDataType.INTEGER, section.getInt("value"))
```

You can also modify the texture pack in a similar way:

```java
ResourcePack.addModifiers(packFolder -> {/* your modifications */});
```
{% endhint %}

Finally register your mechanic!

To summarize the tutorial, [here is the complete source code of the durability mechanics](https://github.com/Th0rgal/Oraxen/blob/master/src/main/java/io/th0rgal/oraxen/items/mechanics/provided/durability/DurabilityMechanicFactory.java).
