---
title:  "Queen City Hacks Talk - Javimmutable Jackson Module"
date:   2018-01-07 13:00:00 -0500
categories: queencityhacks
---
# What I did last month...

## Javimmutable Collections

- Released Javimmutable 2.2.0
    - https://github.com/brianburton/java-immutable-collections
    - https://brianburton.github.io/javimmutable/2017/12/23/jimmutables-2.2.0.html
    - New tree map
    - New hash map
    - All collections serializable (apache spark friendly)
    - All collections produce collectors for streams
- Most important features now complete
- Future development
    - improve spliterators
    - integrations with other common libraries
    - [everywhere](http://m.memegen.com/ku6don.jpg)
    

## JSON Support

- https://github.com/brianburton/javimmutable-jackson
- Round trip JSON serialization using Jackson (https://github.com/FasterXML/jackson)
- Code from Guava's Jackson module very informative.
    - https://github.com/FasterXML/jackson-datatypes-collections
    - [Never try to...](http://m.memegen.com/acnz1l.jpg)

#### JSON Sample

````
{
    "cities": [
        "oxford",
        "cambridge"
    ],
    "names": {
        "names": [
            "jones",
            "smith",
            null
        ]
    }
}
````

#### Mutable Sample

Normal mutable bean containing a list of strings.
- No constructor needed
- Setters used to set the properties from JSON
- @JsonProperty annotation used to identify properties
- Suffers from all the problems related to mutability
    - unsafe to share
    - caller could change List after calling setCities()
    - No guarantee all fields are initialized

````
    public class OrgBean
    {
        @JsonProperty
        private NamesBean names;
        @JsonProperty
        private List<String> cities;

        public NamesBean getNames()
        {
            return names;
        }

        public void setNamesBean(NamesBean value)
        {
            names = value;
        }
        
        public List<String> getCities()
        {
            return cities;
        }
        
        public void setCities(List<String> value)
        {
            cities = value;
        }
    }
````

#### Immutable Sample

Immutable version of bean
- Single constructor to initialize all fields.
- All fields final
- No setters
- No public constructor
- All the benefits of immutability
    - safe to share
    - cities can be safely retained and caller can't change it
- @JsonCreator annotation tells Jackson how to construct object.
- @JsonProperty annotations identify which properties map to which parameters.

````
    public class OrgBean
    {
        private final NamesBean names;
        private final JImmutableList<String> cities;

        @JsonCreator
        public OrgBean(@JsonProperty("names") NamesBean names,
                       @JsonProperty("cities") JImmutableList<String> cities)
        {
            this.names = names;
            this.cities = cities;
        }

        public NamesBean getNames()
        {
            return names;
        }

        public JImmutableList<String> getCities()
        {
            return cities;
        }
    }

    public static class NamesBean
    {
        private final JImmutableList<String> names;

        @JsonCreator
        public NamesBean(@JsonProperty("names") JImmutableList<String> names)
        {
            this.names = names;
        }

        public JImmutableList<String> getNames()
        {
            return names;
        }
    }

````

#### Module

Can extend Jackson to support Javimmutable collections using a module.

````
public class JImmutableModule
    extends Module
{
    @Override
    public void setupModule(SetupContext context)
    {
        context.addDeserializers(new JImmutableDeserializers());
        context.addSerializers(new JImmutableSerializers());
        context.addTypeModifier(new JImmutableTypeModifier());
    }
}
````

Which is used like this:

````
    ObjectMapper mapper = new ObjectMapper();
    mapper.registerModules(new JImmutableModule());
    ...
    NamesBean names = new NamesBean(list("jones", "smith", null));
    OrgBean org = new OrgBean(names, list("oxford", "cambridge"));
    String json = mapper.writeValueAsString(org);
    ...
    OrgBean readBack = mapper.readValue(json, OrgBean.class);

````

#### Serializer

Writing the serializer is surprisingly easy.
- Jackson comes with support for *collection like* data types.
- Jackson deduces all of the types involved and passes us classes to do the serialization of values.  We just have to decide how to serialize the collection.
- Jackson provides `IterableSerializer` class to serialize anything that's `Iterable`.

````
public class JImmutableSerializers
    extends Serializers.Base
{
    @Override
    public JsonSerializer<?> findCollectionLikeSerializer(SerializationConfig config,
                                                          CollectionLikeType type,
                                                          BeanDescription beanDesc,
                                                          TypeSerializer elementTypeSerializer,
                                                          JsonSerializer<Object> elementValueSerializer)
    {
        if (type.isTypeOrSubTypeOf(JImmutableList.class) || type.isTypeOrSubTypeOf(JImmutableSet.class)) {
            return new IterableSerializer(type.getContentType(), false, elementTypeSerializer);
        }
        return super.findCollectionLikeSerializer(config, type, beanDesc, elementTypeSerializer, elementValueSerializer);
    }
}
````

#### Deserializer

Writing the deserializer is much more complicated.
- Have to use a callback to get more info from Jackson before deserializing.
- Have to take care of recognizing tokens in the input stream.
- Have to deal with some Jackson configuration options that user might have selected.

This is a class I wrote to deserialize any immutable collection that implements `Insertable` interface.

````
public class InsertableDeserializer<T extends Insertable>
    extends StdDeserializer<T>
    implements ContextualDeserializer
{
    private final CollectionLikeType collectionType;
    private final JsonDeserializer valueDeserializer;
    private final TypeDeserializer typeDeserializer;
    private final boolean acceptSingleValue;
    private final T empty;

    public InsertableDeserializer(CollectionLikeType collectionType,
                                  JsonDeserializer valueDeserializer,
                                  TypeDeserializer typeDeserializer,
                                  boolean acceptSingleValue,
                                  T empty)
    {
        super(collectionType);
        this.collectionType = collectionType;
        this.valueDeserializer = valueDeserializer;
        this.typeDeserializer = typeDeserializer;
        this.acceptSingleValue = acceptSingleValue;
        this.empty = empty;
    }

    @Override
    public JsonDeserializer<?> createContextual(DeserializationContext context,
                                                BeanProperty property)
        throws JsonMappingException
    {
        JsonDeserializer<?> valueDeserializer = this.valueDeserializer;
        if (valueDeserializer == null) {
            valueDeserializer = context.findContextualValueDeserializer(collectionType.getContentType(), property);
        }

        TypeDeserializer typeDeserializer = this.typeDeserializer;
        if (typeDeserializer != null) {
            typeDeserializer = typeDeserializer.forProperty(property);
        }

        boolean acceptSingleValue = context.isEnabled(DeserializationFeature.ACCEPT_SINGLE_VALUE_AS_ARRAY);
        return new InsertableDeserializer<>(collectionType, valueDeserializer, typeDeserializer, acceptSingleValue, empty);
    }

    @Override
    public T deserialize(JsonParser parser,
                         DeserializationContext context)
        throws IOException, JsonProcessingException
    {
        if (parser.isExpectedStartArrayToken()) {
            return deserializeArrayValues(parser, context);
        } else if (acceptSingleValue) {
            return deserializeSingleValue(parser, context);
        } else {
            throw context.mappingException(collectionType.getRawClass());
        }
    }

    @SuppressWarnings("unchecked")
    private T deserializeSingleValue(JsonParser parser,
                                     DeserializationContext context)
        throws IOException
    {
        Object value = deserializeToken(parser, context, parser.getCurrentToken());
        return (T)empty.insert(value);
    }

    @SuppressWarnings("unchecked")
    private T deserializeArrayValues(JsonParser parser,
                                     DeserializationContext context)
        throws IOException
    {
        T result = empty;

        JsonToken token;
        while ((token = parser.nextToken()) != JsonToken.END_ARRAY) {
            Object value = deserializeToken(parser, context, token);
            result = (T)result.insert(value);
        }

        return result;
    }

    private Object deserializeToken(JsonParser parser,
                                    DeserializationContext context,
                                    JsonToken token)
        throws IOException
    {
        Object value;
        if (token == JsonToken.VALUE_NULL) {
            value = null;
        } else if (typeDeserializer == null) {
            value = valueDeserializer.deserialize(parser, context);
        } else {
            value = valueDeserializer.deserializeWithType(parser, context, typeDeserializer);
        }
        return value;
    }
}
````

This is the deserializer I wrote that creates appropriately configured `InsertableDeserializer` based on collection type.

````
public class JImmutableDeserializers
    extends Deserializers.Base
{
    @Override
    public JsonDeserializer<?> findCollectionLikeDeserializer(CollectionLikeType type,
                                                              DeserializationConfig config,
                                                              BeanDescription beanDesc,
                                                              TypeDeserializer elementTypeDeserializer,
                                                              JsonDeserializer<?> elementDeserializer)
        throws JsonMappingException
    {
        if (type.isTypeOrSubTypeOf(Insertable.class)) {
            if (type.isTypeOrSubTypeOf(JImmutableRandomAccessList.class)) {
                return new InsertableDeserializer<>(type, elementDeserializer, elementTypeDeserializer, false, JImmutables.ralist());
            } else if (type.isTypeOrSubTypeOf(JImmutableList.class)) {
                return new InsertableDeserializer<>(type, elementDeserializer, elementTypeDeserializer, false, JImmutables.list());
            } else if (type.isTypeOrSubTypeOf(JImmutableSet.class)) {
                if (type.isTypeOrSubTypeOf(SortedOrderSet.class)) {
                    requireCollectionOfComparableElements(type);
                    return new InsertableDeserializer<>(type, elementDeserializer, elementTypeDeserializer, false, JImmutables.sortedSet());
                } else if (type.isTypeOrSubTypeOf(InsertOrderSet.class)) {
                    return new InsertableDeserializer<>(type, elementDeserializer, elementTypeDeserializer, false, JImmutables.insertOrderSet());
                } else {
                    return new InsertableDeserializer<>(type, elementDeserializer, elementTypeDeserializer, false, JImmutables.set());
                }
            }
            throw new IllegalArgumentException("Class is not supported: " + type.getRawClass().getName());
        }
        return super.findCollectionLikeDeserializer(type, config, beanDesc, elementTypeDeserializer, elementDeserializer);
    }

    private void requireCollectionOfComparableElements(CollectionLikeType actualType)
    {
        Class<?> elemType = actualType.getContentType().getRawClass();
        if (!Comparable.class.isAssignableFrom(elemType)) {
            throw new IllegalArgumentException("Can not handle JImmutables.sortedSet() with elements that are not Comparable<?> (" + elemType.getName() + ")");
        }
    }
}
````

#### What's Next

- Test to verify that module works with multiple versions of javimmutable collections.
    - All collections created using standard factory methods in `JImmutables` class.
    - Compiled with Java 7 compiler so it can be used with Java 7 or 8.
- Might add support for other collection types (maps, linked lists, multi-sets, list-maps, set-maps, etc).
    - Some of these are VERY complicated.

#### Final thoughts

[Json from immutable collections](http://m.memegen.com/e7ivq8.jpg)

## Get the word out

- I want users!
- Ideas
    - Define contribution guidelines for project
        - would make users more likely to contribute
    - Other subreddits
        - programming
        - cool opensource projects
        - comp sci
