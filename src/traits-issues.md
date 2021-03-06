# Traits: issues, comments, todo's... #

If we are completely transparent about the representation of traits as property maps, then traits.js truly becomes _both_ a general-purpose property map combinator library _and_ a principled approach to trait-based object composition.

- Trait.create could be enhanced to enable sharing of structure between instances.
Method props could be separated
Instantiated objects contain only data props and delegate to the shared method props.
need a "trait cache": maps trait instances to their shared object
on cache miss, Trait.create builds the shared object and stores it in the cache

two problems:
- since prototype of the object must now be the 'common' object, this interferes with the first arg to Trait.create
(could enable optimization only if proto = Object.prototype, or extend cache to work on 2 keys: prototype + trait)
- the 'this' variable in the common object cannot be bound. Must be left unbound such that 'this' can refer to the per-instance sub-object. Problem: this object is accessible from the trait instance via Object.getPrototypeOf
this would not be a problem for open:true objects.

## Extensible Annotations ##

```js
Trait.transform = function(pdm, transformations) {
  var props = {};
  for (var prop in pdm) {// only own props
    var pd = pdm[name];
    for (var attr in pd) { // only own props
      // filter out ES5 attributes?
      if (attr in transformations) {
        props[name] = transformations[attr](name,pd); // each transform is a fn(pd) -> pd
      }
    }
  }
  return props;
};

var traitRules = Object.getOwnProperties({
  required: function(name,pd) { if (!name in proto) { throw new Error('required: '+name); } },
  conflict: function(name,pd) { throw new Error('conflict: '+name); },
  method: function(name, pd) {
    // ugly imperative, should probably construct a new prop descriptor
    if (pd.value) { pd.value = freezeAndBind(pd.value); }
    if (pd.get) { pd.get = freezeAndBind(pd.get, self); } // TODO: how to get self???
    if (pd.set) { pd.set = freezeAndBind(pd.set, self); } // TODO: how to get self???
    return pd;
  }
});

Trait.create = function(proto, pdm, additionalRules) {
  additionalRules = additionalRules || {};
  return Object.freeze(
           Object.create(proto,
             Trait.transform(pdm,
               Object.create(additionalRules, traitRules))));
};
```

## Design rationale not pursued ##

- rename 'open' option to 'final'?
final has the advantage that:
- it's already known as a keyword enforcing similar semantics
- it explicitly tells the programmer that this object will not play nice with objects that delegate to it
the downside: if 'final:true' is the default, then usually 'final' will show up in the source when one means to write: 'final:false', and cognitively it's bad to introduce 'final' only when you create non-final objects...
so: final:false as default could work, but then the 'unsafe' alternative is the default...


- Trait.object({...}) -> would make more sense for users just to write object({...}). Bring 'object' into importer's scope by default? Maybe not... keep it clean... importer can always write var object = Trait.object;


- alternative API: represent traits as function objects instead of as property maps (currently not pursued, disadvantages outweigh advantages)
if we would represent traits as functions, we could use the new operator or function calls to instantiate traits.
We could still provide a to_property_map(trait) -> pdmap function as well.

We could also refactor compose, resolve, override to be defined as methods on the trait function objects:

```js
var T = trait({...});
var T3 = T.compose(T2);
var o = new T3({ extend: proto }); // or T3(options) without new
```

Pro:
- methods do not pollute an importer's namespace (only need to make available a 'trait' constructor, all other methods are defined on traits)
- we could use a method named 'new' to write trait.new(options) rather than 'build(trait,options)'.
- we can write 'new trait(options)', such that trait instantiation is more close to regular object instantiation in JS.

Con:
- methods make the composition operators look asymmetric as they seem to 'prioritize' the first argument. Might be OK if we rename them and remove variable arity: t1.composeWith(t2), t1.overrideWith(t2), t1.resolve({...}), t1.new(options), t1.eqv(t2)
- traits are no longer property descriptor maps. We need conversion functions to convert between traits and pdmaps:
asTrait(pdMap) -> trait
trait.toPropertyDescriptorMap() -> pdMap
- exposing traits as constructors may only cause additional confusion. Also, stateful traits may cause confusion in combination with 'new', e.g. 'new makePointTrait(x,y)' is wrong! Should be: 'new (makePointTrait(x,y))(options)' or simply '(makePointTrait(x,y))(options)'
- we lose the analogy between Trait.create(proto, pdmap) and Object.create(proto, pdmap)