---
title:  "Step 5: Create tables and entities with Jetbrains Exposed framework" 
layout: post
categories: setup exposed introductory
--- 
Next challenge: let’s prepare a database that will store all our recipes. We’ll use a Jetbrains Exposed ORM framework coupled with a postgres SQL for it.

First, add necessary dependencies for Exposed to your project.
Add the exposed maven repository to your project level `build.gradle.kts`:

{% highlight kotlin %}
allprojects {
   repositories {
       google()
       mavenCentral()
       maven("https://dl.bintray.com/kotlin/exposed")
   }
}
{% endhighlight %}

In you `Versions.kt` declare relevant dependencies:

{% highlight kotlin %}
object Versions {
const val EXPOSED_VERSION = "0.36.2"
. . .
object Jvm {
const val JETBRAINS_EXPOSED_CORE = "org.jetbrains.exposed:exposed-core:$EXPOSED_VERSION"
const val JETBRAINS_EXPOSED_DAO = "org.jetbrains.exposed:exposed-dao:$EXPOSED_VERSION"
const val JETBRAINS_EXPOSED_JDBC = "org.jetbrains.exposed:exposed-jdbc:$EXPOSED_VERSION"
}
{% endhighlight %}

And then use them in your backend module: 

{% highlight kotlin %}
implementation(Versions.Jvm.JETBRAINS_EXPOSED_CORE)
implementation(Versions.Jvm.JETBRAINS_EXPOSED_DAO)
implementation(Versions.Jvm.JETBRAINS_EXPOSED_JDBC)
{% endhighlight %}

Don’t forget to sync your project.
Now that all dependencies are implemented, let’s get down to designing our database.

Our SQL database will have three tables: `Ingredients`, `Instructions` and `Recipes`. Every Entity in Exposed DAO has a table which extends `IntIdTable` and defines the relationships between variables. Then we can use those relationships to create `Entities` and transform them into classes which we can later use in our application. 

First, let’s create tables for each class. Add a new package called `storage` to your backend package. Then add a package called `exposed` inside the storage package. Prepare your project structure for the changes to look like this:

<img src="{{site.baseurl}}/assets/images/step-5/1.png" alt="add exposed package" width="350"/>

Then add the following tables in relevant files and perform necessary import from `org.jetbrains.exposed` along the way:

{% highlight kotlin %}
internal object IngredientTable : IntIdTable("ingredients") { [1]
   val name = varchar("name", 100) [2]
   val amount = decimal("amount", 2, 1) [3]
   val metric = varchar("metric", 100)
   val recipe = reference( [4]
       "recipe_id",
       RecipeTable,
       onDelete = ReferenceOption.CASCADE,
       onUpdate = ReferenceOption.CASCADE
   ) [5]
} 

internal object InstructionTable : IntIdTable("instructions") {
   val order = integer("order") [3]
   val description = varchar("description", 100)
   val recipe = reference(
       "recipe_id",
       RecipeTable,
       onDelete = ReferenceOption.CASCADE,
       onUpdate = ReferenceOption.CASCADE
   )
}

internal object RecipeTable : IntIdTable("recipes") {
   val title = varchar("title", 100) 
}
{% endhighlight %}

[1] We create a table which gets Integer ids automatically and name it “ingredients”. This table holds such properties as name, amount, metric (like in our data class that we defined before). Similarly, the instruction table holds previously defined properties. Recipe table only has a title (read on to find out what to do with instructions and ingredients).<br>
[2] When we need to create a text property, we use varchar. <br>
[3] For decimal/double values we use decimal and integer for integer values.<br>
[4] Now, our recipes actually should have a list of ingredients and instructions. In this simplified database every “ingredient” and “instruction” comes up only in one recipe, but a recipe has a number of ingredients and instructions. Therefore we define a reference in our “ingredients” and “instructions” tables that will point to the recipe where they are used.<br>
[5] onDelete and onCascade properties tell us how these entries in the table will behave if our parent gets removed. In this case, if the recipe is deleted, the ingredients and instructructions that reference that recipe also get deleted.<br>

Now we can build our entities based on these relations. Add the following code to `IngredientEntity.kt`:

{% highlight kotlin %}
class IngredientEntity(id: EntityID<Int>) : IntEntity(id) { [1]
   companion object : IntEntityClass<IngredientEntity>(IngredientTable)

   var name by IngredientTable.name
   var amount by IngredientTable.amount
   var metric by IngredientTable.metric
   var recipe by RecipeEntity referencedOn IngredientTable.recipe [2]

}

fun IngredientEntity.toIngredient() = Ingredient( [3]
   id.value.toLong(),
   name,
   amount.toDouble(),[4]
   metric
)
{% endhighlight %}

[1] Create an Entity based on the relations in IngredientTable. The Long id property is generated automatically.<br>
[2] Here we specify the Recipe Entity where the instruction comes up, using a reference to the Recipe table that we’ve created<br>
[3] Now we can map our database Entity to our class that we defined
   earlier so that we can later use it in the application<br>
[4] The amount is stored as BigDecimal in the database, but we want to use the Double, so we need to convert it<br>

As you can see, now our data classes don’t really match with the entities because of ids. Let’s complete our data classes with ids as follows: 

{% highlight kotlin %}
data class Recipe(
    val id: Long = 0,
    val title: String,
    val ingredients: List<Ingredient>,
    val instructions: List<Instruction>
)
{% endhighlight %}

Add an id to `Ingredient` and `Instruction` as well.

{% highlight kotlin %}
data class Ingredient(
    val id: Long = 0,
    val name: String,
    val amount: Double,
    val metric: String
)

data class Instruction(
    val id: Long = 0,
    val order: Int,
    val description: String,
)
{% endhighlight %}

Do the same procedure for Instructions. Don’t worry if some things are not resolved at first, we’ll get back to them later.

{% highlight kotlin %}
class InstructionEntity(id: EntityID<Int>) : IntEntity(id) { 
   companion object : IntEntityClass<InstructionEntity>(InstructionTable)

   var order by InstructionTable.order
   var description by InstructionTable.description
   var recipe by RecipeEntity referencedOn InstructionTable.recipe 
}

fun InstructionEntity.toInstruction() = Instruction( 
   id.value.toLong(),
   order,
   description 
)
{% endhighlight %}

We only need to create RecipeEntity itself now. Here we’ll need to use those relationships that we’ve defined in Ingredients and Instructions. 

{% highlight kotlin %}
class RecipeEntity(id: EntityID<Int>) : IntEntity(id) { 
   companion object : IntEntityClass<RecipeEntity>(RecipeTable)

   var title by RecipeTable.title
   val ingredients by IngredientEntity referrersOn IngredientTable.recipe [1]
   val instructions by InstructionEntity referrersOn InstructionTable.recipe
}

fun RecipeEntity.toRecipe() = Recipe(
   id.value.toLong(),
   title,
   ingredients.map{it.toIngredient()}, [2]
   instructions.map{it.toInstruction()}
)
{% endhighlight %}

[1] Here we collect all the ingredients and instructions that reference this recipe and create a `SizedIterable<IngredientEntity>` or `SizedIterable<InstructionEntity>`<br>
[2] When transforming our entity to a data class, we need to map our `SizedInterable` to a `List` <br>

Fix missing imports in all tables and entities, if there are any left - this will make all previous unresolved references go away.

So far so good! We’ve made the necessary preparations to set up our local database.<br> In the next step we’ll proceed with postgresSQL.

