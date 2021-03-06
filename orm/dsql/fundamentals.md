[Home](../readme.md "Home")

#### SECTION 2
----
## Fundamentals of DSQL

To understand DSQL you should realize that it works in a very similar way to how other objects in Agile Toolkit working. This consists of two parts:

1. Query parameters are set and are saved inside a property.
2. Query is rendered, parameters are converted and query is executed.

Here is the example of a sample query template:

    select [myexpr]
    
DSQL can have only one template at a time. Calling some methods will also change the template. You can explicitly specify template using `expr()`.

Before query is executed, DSQL renders the template. It searches for the regions in brackets and executes method `render_<text in brackets>`

    function render_myexpr(){
        return 'now()';
    }
    
now to try and put this all together you would need to use this code:

    class MyDSQL extends DB_dsql {
        function render_myexpr(){
            return 'now()';
        }
    }
    
    $this->api->dbConnect();
    $q = $this->api->db->dsql('MyDSQL');
    $q->expr('select [myexpr]');
    echo $q->getOne();

In this example, we define our own field renderer and then reference in the expression. The resulting query should output a current time to you.

### DSQL Argument Types

When you build the query by calling methods, you arguments could be:

1. Field Names, such as \`id\`
2. Unsafe variables, such as :a
3. Expressions such as "`now()`"

-

    $q=$this->api->db->dsql()->table('user');
    $q->where('id',$_GET['id']);
    $q->where('expired','<',$q->expr('now()'));
    echo $q->field('count(*)')->getOne();

The example above will produce query:

    select count(*) from `id`=123 and `expired`<now();
    
Value from `$_GET['id']` is un-safe and would be parametrized to avoid injection. "`now()`" however is an expression and should be left as it is.

It is a job for the renderer to properly recognize the arguments and use one of the wrappers for them. Let's review the previous example and see how to use different escaping techniques:

    class MyDSQL extends DB_dsql {
        function render_test1(){ return 123; }
        function render_test2(){ return $this->bt(123); }
        function render_test3(){ return $this->expr(123); }
        function render_test5(){ return $this->escape(123); }
        function render_test6(){ return $this->consume(123); }
    }
    
Those are all available escaping mechanisms:

1. `bt()` will surround the value with backticks. This is most useful for field and table names, to avoid conflicts when table or field name is same as keyword `("select * from order")`
2. `expr()` will clone DSQL object and set it's template to first argument. DSQL may surround expressions with brackets, but it will not escape them.
3. `escape()` will convert argument into parametric value and will substitute it with sequential identifier such as `:a1, :a2, :a3` etc.
4. Finally, `consume()` will try to identify object correctly and adds support for non-SQL objects such as Model Fields.

### Exercise

*This exercise is only for your understanding. For normal usage of DSQL you don't need to do this.*

Let's review how the "set" operator works in "update" query. First, let's use the following simplified template:

    update [table] set [set]

To help users interface with our template, we must have the following two methods:

    funciton table($table){
        $this->args['table']=$table;
        return $this;
    }
    
    function set($field,$value){
        $this->args['set'][]=array($field,$value);
        return $this;
    }
    
It's important that you don't use any escaping mechanisms on the arguments just yet. They may refer to expression which can still be modified from outside. The arguments are packed into an internal property "args". Next, let's review the rendering part of the arguments. This time I'll be using different escaping mechanisms in different situations.

    funciton render_table(){
        return $this->bt($this->args['table']);
    }
    function render_set(){
        $result=array();
        foreach($this->args['set'] as list($field,$value)){
            $field=$this->bt($field);
            if(is_object($value)){
                $value=$this->consume($value);
            }else{
                $value=$this->escape($value);
            }
            $result[]=$field.'='.$value;
        }
        return join(', ',$result);
    }

Table would need to be back-ticked and we don't really need to worry about expressions. For the "set" rendering things are bit more complex. We allow multiple calls to `set()` and then we need to produce the equation for each field and join result by commas. The first argument, the field, needs to be back-ticked. Second argument may be an object, but if it's now, it most probably would contain an unsafe parameter, so we use `escape()` to convert it into parametric value.

Consume here would recursively render the expression and join the parameters from the subquery to our own. In some situations we would need to surround consume with a brackets, if SQL syntax requires it.

NOTE: This exercise is mentioned here only to help you understand DSQL. You must not try to re-implement rendering of table or set arguments as it is already implemented properly in DSQL. Let's take look at other templates DSQL supports.

[Next Section](section3.md "Next Section")