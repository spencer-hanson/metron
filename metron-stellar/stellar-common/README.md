

# Contents

* [Stellar Language](#stellar-language)
    * [Stellar Language Keywords](#stellar-language-keywords)
    * [Stellar Core Functions](#stellar-core-functions)
    * [Stellar Benchmarks](#stellar-benchmarks)
    * [Stellar Shell](#stellar-shell)
* [Stellar Configuration](#stellar-configuration)


# Stellar Language

For a variety of components (threat intelligence triage and field
transformations) we have the need to do simple computation and
transformation using the data from messages as variables.  
For those purposes, there exists a simple, scaled down DSL 
created to do simple computation and transformation.

The query language supports the following:
* Referencing fields in the enriched JSON
* String literals are quoted with either `'` or `"`
* String literals support escaping for `'`, `"`, `\t`, `\r`, `\n`, and backslash 
  * The literal `'\'foo\''` would represent `'foo'`
  * The literal `"\"foo\""` would represent `"foo"`
  * The literal `'foo \\ bar'` would represent `foo \ bar`
* Simple boolean operations: `and`, `not`, `or`
* Simple arithmetic operations: `*`, `/`, `+`, `-` on real numbers or integers
* Simple comparison operations `<`, `>`, `<=`, `>=`
* Simple equality comparison operations `==`, `!=`
* if/then/else comparisons (i.e. `if var1 < 10 then 'less than 10' else '10 or more'`)
* Determining whether a field exists (via `exists`)
* An `in` operator that works like the `in` in Python
* The ability to have parenthesis to make order of operations explicit
* User defined functions, including Lambda expressions 

## Stellar Language Keywords
The following keywords need to be single quote escaped in order to be used in Stellar expressions:

|               |               |             |             |             |
| :-----------: | :-----------: | :---------: | :---------: | :---------: |
| not           | else          | exists      | if          | then        |
| and           | or            | in          | ==          | !=          |
| \<=           | \>            | \>=         | \+          | \-          |
| \<            | ?             | \*          | /           | ,           |

Using parens such as: "foo" : "\<ok\>" requires escaping; "foo": "\'\<ok\>\'"

## Stellar Language Inclusion Checks (`in` and `not in`)
1. `in` supports string contains. e.g. `'foo' in 'foobar' == true`
2. `in` supports collection contains. e.g. `'foo' in [ 'foo', 'bar' ] == true`
3. `in` supports map key contains. e.g. `'foo' in { 'foo' : 5} == true`
4. `not in` is the negation of the in expression. e.g. `'grok' not in 'foobar' == true`

## Stellar Language Comparisons (`<`, `<=`, `>`, `>=`)

1. If either side of the comparison is null then return false.
2. If both values being compared implement number then the following:
    * If either side is a double then get double value from both sides and compare using given operator.
    * Else if either side is a float then get float value from both sides and compare using given operator.
    * Else if either side is a long then get long value from both sides and compare using given operator.
    * Otherwise get the int value from both sides and compare using given operator.
3. If both sides are of the same type and are comparable then use the compareTo method to compare values.
4. If none of the above are met then an exception is thrown.

## Stellar Language Equality Check (`==`, `!=`)

Below is how the `==` operator is expected to work:

1. If either side of the expression is null then check equality using Java's `==` expression.
2. Else if both sides of the expression are of Java's type Number then:
   * If either side of the expression is a double then use the double value of both sides to test equality.
   * Else if either side of the expression is a float then use the float value of both sides to test equality.
   * Else if either side of the expression is a long then use long value of both sides to test equality.
   * Otherwise use int value of both sides to test equality
3. Otherwise use equals method compare the left side with the right side.

The `!=` operator is the negation of the above.

## Stellar Language Lambda Expressions

Stellar provides the capability to pass lambda expressions to functions
which wish to support that layer of indirection.  The syntax is:
* `(named_variables) -> stellar_expression` : Lambda expression with named variables
  * For instance, the lambda expression which calls `TO_UPPER` on a named argument `x` could be be expressed as `(x) -> TO_UPPER(x)`.
* `var -> stellar_expression` : Lambda expression with a single named variable, `var`
  * For instance, the lambda expression which calls `TO_UPPER` on a named argument `x` could be expressed as `x -> TO_UPPER(x)`.  Note, this is more succinct but equivalent to the example directly above.
* `() -> stellar_expression` : Lambda expression with no named variables.
  * If no named variables are needed, you may omit the named variable section.  For instance, the lambda expression which returns a constant `false` would be `() -> false`

where 
* `named_variables` is a comma separated list of variables to use in the Stellar expression
* `stellar_expression` is an arbitrary stellar expression


In the core language functions, we support basic functional programming primitives such as
* `MAP` - Applies a lambda expression over a list of input.  For instance `MAP([ 'foo', 'bar'], (x) -> TO_UPPER(x) )` returns `[ 'FOO', 'BAR' ]`
* `FILTER` - Filters a list by a predicate in the form of a lambda expression.  For instance `FILTER([ 'foo', 'bar'], (x ) -> x == 'foo' )` returns `[ 'foo' ]`
* `REDUCE` - Applies a function over a list of input.  For instance `REDUCE([ 1, 2, 3], (sum, x) -> sum + x, 0 )` returns `6`

## Stellar Core Functions

|                                                                                                    |
| ----------                                                                                         |
| [ `ABS`](../../metron-analytics/metron-statistics#abs)                                             |
| [ `APPEND_IF_MISSING`](#append_if_missing)                                                         |
| [ `BIN`](../../metron-analytics/metron-statistics#bin)                                             |
| [ `BLOOM_ADD`](#bloom_add)                                                                         |
| [ `BLOOM_EXISTS`](#bloom_exists)                                                                   |
| [ `BLOOM_INIT`](#bloom_init)                                                                       |
| [ `BLOOM_MERGE`](#bloom_merge)                                                                     |
| [ `CHOP`](#chop)                                                                                   |
| [ `CHOMP`](#chomp)                                                                                 |
| [ `COUNT_MATCHES`](#count_matches)                                                                 |
| [ `DAY_OF_MONTH`](#day_of_month)                                                                   |
| [ `DAY_OF_WEEK`](#day_of_week)                                                                     |
| [ `DAY_OF_YEAR`](#day_of_year)                                                                     |
| [ `DOMAIN_REMOVE_SUBDOMAINS`](#domain_remove_subdomains)                                           |
| [ `DOMAIN_REMOVE_TLD`](#domain_remove_tld)                                                         |
| [ `DOMAIN_TO_TLD`](#domain_to_tld)                                                                 |
| [ `ENDS_WITH`](#ends_with)                                                                         |
| [ `ENRICHMENT_EXISTS`](#enrichment_exists)                                                         |
| [ `ENRICHMENT_GET`](#enrichment_get)                                                               |
| [ `FILL_LEFT`](#fill_left)                                                                         |
| [ `FILL_RIGHT`](#fill_right)                                                                       |
| [ `FILTER`](#filter)                                                                         |
| [ `FORMAT`](#format)                                                                               |
| [ `HLLP_CARDINALITY`](../../metron-analytics/metron-statistics#hllp_cardinality)                   |
| [ `HLLP_INIT`](../../metron-analytics/metron-statistics#hllp_init)                                 |
| [ `HLLP_MERGE`](../../metron-analytics/metron-statistics#hllp_merge)                               |
| [ `HLLP_OFFER`](../../metron-analytics/metron-statistics#hllp_offer)                               |
| [ `GEO_GET`](#geo_get)                                                                             |
| [ `GET`](#get)                                                                                     |
| [ `GET_FIRST`](#get_first)                                                                         |
| [ `GET_LAST`](#get_last)                                                                           |
| [ `IN_SUBNET`](#in_subnet)                                                                         |
| [ `IS_DATE`](#is_date)                                                                             |
| [ `IS_DOMAIN`](#is_domain)                                                                         |
| [ `IS_EMAIL`](#is_email)                                                                           |
| [ `IS_EMPTY`](#is_empty)                                                                           |
| [ `IS_INTEGER`](#is_integer)                                                                       |
| [ `IS_IP`](#is_ip)                                                                                 |
| [ `IS_URL`](#is_url)                                                                               |
| [ `JOIN`](#join)                                                                                   |
| [ `KAFKA_GET`](#kafka_get)                                                                         |
| [ `KAFKA_PROPS`](#kafka_props)                                                                     |
| [ `KAFKA_PUT`](#kafka_put)                                                                         |
| [ `KAFKA_TAIL`](#kafka_tail)                                                                       |
| [ `LENGTH`](#length)                                                                               |
| [ `LIST_ADD`](#list_add)                                                                               |
| [ `MAAS_GET_ENDPOINT`](#maas_get_endpoint)                                                         |
| [ `MAAS_MODEL_APPLY`](#maas_model_apply)                                                           |
| [ `MAP`](#map)                                                                       |
| [ `MAP_EXISTS`](#map_exists)                                                                       |
| [ `MONTH`](#month)                                                                                 |
| [ `PREPEND_IF_MISSING`](#prepend_if_missing)                                                       |
| [ `PROFILE_GET`](#profile_get)                                                                     |
| [ `PROFILE_FIXED`](#profile_fixed)                                                                 |
| [ `PROFILE_WINDOW`](#profile_window)                                                               |
| [ `PROTOCOL_TO_NAME`](#protocol_to_name)                                                           |
| [ `REDUCE`](#reduce)                                                                   |
| [ `REGEXP_MATCH`](#regexp_match)                                                                   |
| [ `REGEXP_GROUP_VAL`](#regexp_group_val)                                                                   |
| [ `SPLIT`](#split)                                                                                 |
| [ `STARTS_WITH`](#starts_with)                                                                     |
| [ `STATS_ADD`](../../metron-analytics/metron-statistics#stats_add)                                 |
| [ `STATS_BIN`](../../metron-analytics/metron-statistics#stats_bin)                                 |
| [ `STATS_COUNT`](../../metron-analytics/metron-statistics#stats_count)                             |
| [ `STATS_GEOMETRIC_MEAN`](../../metron-analytics/metron-statistics#stats_geometric_mean)           |
| [ `STATS_INIT`](../../metron-analytics/metron-statistics#stats_init)                               |
| [ `STATS_KURTOSIS`](../../metron-analytics/metron-statistics#stats_kurtosis)                       |
| [ `STATS_MAX`](../../metron-analytics/metron-statistics#stats_max)                                 |
| [ `STATS_MEAN`](../../metron-analytics/metron-statistics#stats_mean)                               |
| [ `STATS_MERGE`](../../metron-analytics/metron-statistics#stats_merge)                             |
| [ `STATS_MIN`](../../metron-analytics/metron-statistics#stats_min)                                 |
| [ `STATS_PERCENTILE`](../../metron-analytics/metron-statistics#stats_percentile)                   |
| [ `STATS_POPULATION_VARIANCE`](../../metron-analytics/metron-statistics#stats_population_variance) |
| [ `STATS_QUADRATIC_MEAN`](../../metron-analytics/metron-statistics#stats_quadratic_mean)           |
| [ `STATS_SD`](../../metron-analytics/metron-statistics#stats_sd)                                   |
| [ `STATS_SKEWNESS`](../../metron-analytics/metron-statistics#stats_skewness)                       |
| [ `STATS_SUM`](../../metron-analytics/metron-statistics#stats_sum)                                 |
| [ `STATS_SUM_LOGS`](../../metron-analytics/metron-statistics#stats_sum_logs)                       |
| [ `STATS_SUM_SQUARES`](../../metron-analytics/metron-statistics#stats_sum_squares)                 |
| [ `STATS_VARIANCE`](../../metron-analytics/metron-statistics#stats_variance)                       |
| [ `STRING_ENTROPY`](#string_entropy)                                                               |
| [ `SYSTEM_ENV_GET`](#system_env_get)                                                               |
| [ `SYSTEM_PROPERTY_GET`](#system_property_get)                                                     |
| [ `TO_DOUBLE`](#to_double)                                                                         |
| [ `TO_EPOCH_TIMESTAMP`](#to_epoch_timestamp)                                                       |
| [ `TO_FLOAT`](#to_float)                                                                           |
| [ `TO_INTEGER`](#to_integer)                                                                       |
| [ `TO_LONG`](#to_long)                                                                             |
| [ `TO_LOWER`](#to_lower)                                                                           |
| [ `TO_STRING`](#to_string)                                                                         |
| [ `TO_UPPER`](#to_upper)                                                                           |
| [ `TRIM`](#trim)                                                                                   |
| [ `URL_TO_HOST`](#url_to_host)                                                                     |
| [ `URL_TO_PATH`](#url_to_path)                                                                     |
| [ `URL_TO_PORT`](#url_to_port)                                                                     |
| [ `URL_TO_PROTOCOL`](#url_to_protocol)                                                             |
| [ `WEEK_OF_MONTH`](#week_of_month)                                                                 |
| [ `WEEK_OF_YEAR`](#week_of_year)                                                                   |
| [ `YEAR`](#year)                                                                                   |
| [ `ZIP`](#zip)                                                                                   |
| [ `ZIP_JAGGED`](#zip_jagged)                                                                                   |

### `APPEND_IF_MISSING`
  * Description: Appends the suffix to the end of the string if the string does not already end with any of the suffixes.
  * Input:
    * string - The string to be appended.
    * suffix - The string suffix to append to the end of the string.
    * additionalsuffix - Optional - Additional string suffix that is a valid terminator.
  * Returns: A new String if prefix was prepended, the same string otherwise.

### `BLOOM_ADD`
  * Description: Adds an element to the bloom filter passed in
  * Input:
    * bloom - The bloom filter
    * value* - The values to add
  * Returns: Bloom Filter
  
### `BLOOM_EXISTS`
  * Description: If the bloom filter contains the value
  * Input:
    * bloom - The bloom filter
    * value - The value to check
  * Returns: True if the filter might contain the value and false otherwise

### `BLOOM_INIT`
  * Description: Returns an empty bloom filter
  * Input:
    * expectedInsertions - The expected insertions
    * falsePositiveRate - The false positive rate you are willing to tolerate
  * Returns: Bloom Filter

### `BLOOM_MERGE`
  * Description: Returns a merged bloom filter
  * Input:
    * bloomfilters - A list of bloom filters to merge
  * Returns: Bloom Filter or null if the list is empty

### `CHOP`
  * Description: Remove the last character from a String
  * Input:
    * string - the String to chop last character from, may be null
  * Returns: String without last character, null if null String input

### `CHOMP`
  * Description: Removes one newline from end of a String if it's there, otherwise leave it alone. A newline is "\n", "\r", or "\r\n"
  * Input:
    * string - the String to chomp a newline from, may be null
  * Returns: String without newline, null if null String input

### `COUNT_MATCHES`
  * Description: Counts how many times the substring appears in the larger string.
  * Input:
    * string - the CharSequence to check, may be null.
    * substring/character - the substring or character to count, may be null.
  * Returns: the number of non-overlapping occurrences, 0 if either CharSequence is null.

### `DAY_OF_MONTH`
  * Description: The numbered day within the month.  The first day within the month has a value of 1.
  * Input:
    * dateTime - The datetime as a long representing the milliseconds since unix epoch
  * Returns: The numbered day within the month.

### `DAY_OF_WEEK`
  * Description: The numbered day within the week.  The first day of the week, Sunday, has a value of 1.
  * Input:
    * dateTime - The datetime as a long representing the milliseconds since unix epoch
  * Returns: The numbered day within the week.

### `DAY_OF_YEAR`
  * Description: The day number within the year.  The first day of the year has value of 1.
  * Input:
    * dateTime - The datetime as a long representing the milliseconds since unix epoch
  * Returns: The day number within the year.

### `DOMAIN_REMOVE_SUBDOMAINS`
  * Description: Removes the subdomains from a domain.
  * Input:
    * domain - Fully qualified domain name
  * Returns: The domain without the subdomains.  (for example, DOMAIN_REMOVE_SUBDOMAINS('mail.yahoo.com') yields 'yahoo.com')

### `DOMAIN_REMOVE_TLD`
  * Description: Removes the top level domain (TLD) suffix from a domain.
  * Input:
    * domain - Fully qualified domain name
  * Returns: The domain without the TLD.  (for example, DOMAIN_REMOVE_TLD('mail.yahoo.co.uk') yields 'mail.yahoo')

### `DOMAIN_TO_TLD`
  * Description: Extracts the top level domain from a domain
  * Input:
    * domain - Fully qualified domain name
  * Returns: The TLD of the domain.  (for example, DOMAIN_TO_TLD('mail.yahoo.co.uk') yields 'co.uk')

### `ENDS_WITH`
  * Description: Determines whether a string ends with a specified suffix
  * Input:
    * string - The string to test
    * suffix - The proposed suffix
  * Returns: True if the string ends with the specified suffix and false if otherwise

### `ENRICHMENT_EXISTS`
  * Description: Interrogates the HBase table holding the simple hbase enrichment data and returns whether the enrichment type and indicator are in the table.
  * Input:
    * enrichment_type - The enrichment type
    * indicator - The string indicator to look up
    * nosql_table - The NoSQL Table to use
    * column_family - The Column Family to use
  * Returns: True if the enrichment indicator exists and false otherwise

### `ENRICHMENT_GET`
  * Description: Interrogates the HBase table holding the simple hbase enrichment data and retrieves the tabular value associated with the enrichment type and indicator.
  * Input:
    * enrichment_type - The enrichment type
    * indicator - The string indicator to look up
    * nosql_table - The NoSQL Table to use
    * column_family - The Column Family to use
  * Returns: A Map associated with the indicator and enrichment type.  Empty otherwise.

### `FILL_LEFT`
  * Description: Fills or pads a given string with a given character, to a given length on the left
  * Input:
    * input - string
    * fill - the fill character
    * len - the required length
  * Returns: the filled string

### `FILL_RIGHT`
  * Description: Fills or pads a given string with a given character, to a given length on the right
  * Input:
    * input - string
    * fill - the fill character string
    * len - the required length
  * Returns: Last element of the list

### `FILTER`
  * Description: Applies a filter in the form of a lambda expression to a list. e.g. `FILTER( [ 'foo', 'bar' ] , (x) -> x == 'foo')` would yield `[ 'foo']`
  * Input:
    * list - List of arguments.
    * predicate - The lambda expression to apply.  This expression is assumed to take one argument and return a boolean.
  * Returns: The input list filtered by the predicate.

### `FORMAT`
  * Description: Returns a formatted string using the specified format string and arguments. Uses Java's string formatting conventions.
  * Input:
    * format - string
    * arguments... - object(s)
  * Returns: A formatted string.

### `GEO_GET`
  * Description: Look up an IPV4 address and returns geographic information about it
  * Input:
    * ip - The IPV4 address to lookup
    * fields - Optional list of GeoIP fields to grab. Options are locID, country, city postalCode, dmaCode, latitude, longitude, location_point
  * Returns: If a Single field is requested a string of the field, If multiple fields a map of string of the fields, and null otherwise

### `GET`
  * Description: Returns the i'th element of the list 
  * Input:
    * input - List
    * i - The index (0-based)
  * Returns: First element of the list

### `GET_FIRST`
  * Description: Returns the first element of the list
  * Input:
    * input - List
  * Returns: First element of the list

### `GET_LAST`
  * Description: Returns the last element of the list
  * Input:
    * input - List
  * Returns: Last element of the list

### `IN_SUBNET`
  * Description: Returns true if an IP is within a subnet range.
  * Input:
    * ip - The IP address in string form
    * cidr+ - One or more IP ranges specified in CIDR notation (for example 192.168.0.0/24)
  * Returns: True if the IP address is within at least one of the network ranges and false if otherwise

### `IS_DATE`
  * Description: Determines if the date contained in the string conforms to the specified format.
  * Input:
    * date - The date in string form
    * format - The format of the date
  * Returns: True if the date is in the specified format and false if otherwise.

### `IS_DOMAIN`
  * Description: Tests if a string refers to a valid domain name.  Domain names are evaluated according to the standards RFC1034 section 3, and RFC1123 section 2.1.
  * Input:
    * address - The string to test
  * Returns: True if the string refers to a valid domain name and false if otherwise

### `IS_EMAIL`
  * Description: Tests if a string is a valid email address
  * Input:
    * address - The string to test
  * Returns: True if the string is a valid email address and false if otherwise.

### `IS_EMPTY`
  * Description: Returns true if string or collection is empty or null and false if otherwise.
  * Input:
    * input - Object of string or collection type (for example, list)
  * Returns: True if the string or collection is empty or null and false if otherwise.

### `IS_INTEGER`
  * Description: Determines whether or not an object is an integer.
  * Input:
    * x - The object to test
  * Returns: True if the object can be converted to an integer and false if otherwise.

### `IS_IP`
  * Description: Determine if an string is an IP or not.
  * Input:
    * ip - An object which we wish to test is an ip
    * type (optional) - Object of string or collection type (e.g. list) one of IPV4 or IPV6 or both.  The default is IPV4.
  * Returns: True if the string is an IP and false otherwise.

### `IS_URL`
  * Description: Tests if a string is a valid URL
  * Input:
    * url - The string to test
  * Returns: True if the string is a valid URL and false if otherwise.

### `JOIN`
  * Description: Joins the components in the list of strings with the specified delimiter.
  * Input:
    * list - List of strings
    * delim - String delimiter
  * Returns: String

### `KAFKA_GET`
  * Description: Retrieves messages from a Kafka topic.  Subsequent calls will continue retrieving messages sequentially from the original offset.
  * Input:
    * topic - The name of the Kafka topic.
    * count - The number of Kafka messages to retrieve.
    * config - Optional map of key/values that override any global properties.
  * Returns: List of String

### `KAFKA_PROPS`
  * Description: Retrieves the Kafka properties that are used by other KAFKA_* functions like KAFKA_GET and KAFKA_PUT.  The Kafka properties are compiled from a set of default properties, the global properties, and any overrides.
  * Input:
    * config - An optional map of key/values that override any global properties.
  * Returns: Map of key/value pairs

### `KAFKA_PUT`
  * Description: Sends messages to a Kafka topic.
  * Input:
    * topic - The name of the Kafka topic.
    * messages - A list of messages to write.
    * config - Optional map of key/values that override any global properties.
  * Returns: n/a
  
### `KAFKA_TAIL`
  * Description: etrieves messages from a Kafka topic always starting with the most recent message first.
  * Input:
    * topic - The name of the Kafka topic.
    * count - The number of Kafka messages to retrieve.
    * config - Optional map of key/values that override any global properties.
  * Returns: List of String

### `LENGTH`
  * Description: Returns the length of a string or size of a collection. Returns 0 for empty or null Strings
  * Input:
    * input - Object of string or collection type (e.g. list)
  * Returns: Integer

### `LIST_ADD`
  * Description: Adds an element to a list.
  * Input:
    * list - List to add element to.
    * element - Element to add to list
  * Returns: Resulting list with the item added at the end.
  
### `MAAS_GET_ENDPOINT`
  * Description: Inspects ZooKeeper and returns a map containing the name, version and url for the model referred to by the input parameters.
  * Input:
    * model_name - The name of the model
    * model_version - The optional version of the model.  If the model version is not specified, the most current version is used.
  * Returns: A map containing the name, version, and url for the REST endpoint (fields named name, version and url).  Note that the output of this function is suitable for input into the first argument of MAAS_MODEL_APPLY.

### `MAAS_MODEL_APPLY`
  * Description: Returns the output of a model deployed via Model as a Service. NOTE: Results are cached locally for 10 minutes.
  * Input:
    * endpoint - A map containing the name, version, and url for the REST endpoint
    * function - The optional endpoint path; default is 'apply'
    * model_args - A Dictionary of arguments for the model (these become request params)
  * Returns: The output of the model deployed as a REST endpoint in Map form.  Assumes REST endpoint returns a JSON Map.

### `MAP`
  * Description: Applies lambda expression to a list of arguments. e.g. `MAP( [ 'foo', 'bar' ] , (x) -> TO_UPPER(x) )` would yield `[ 'FOO', 'BAR' ]`
  * Input:
    * list - List of arguments.
    * transform_expression - The lambda expression to apply. This expression is assumed to take one argument.
  * Returns: The input list transformed item-wise by the lambda expression.
  
### `MAP_EXISTS`
  * Description: Checks for existence of a key in a map.
  * Input:
    * key - The key to check for existence
    * map - The map to check for existence of the key
  * Returns: True if the key is found in the map and false if otherwise.

### `MAP_GET`
  * Description: Gets the value associated with a key from a map
  * Input:
    * key - The key
    * map - The map
    * default - Optionally the default value to return if the key is not in the map.
  * Returns: The object associated with the key in the map.  If no value is associated with the key and default is specified, then default is returned. If no value is associated with the key or default, then null is returned.

### `MONTH`
  * Description: The number representing the month.  The first month, January, has a value of 0.
  * Input:
    * dateTime - The datetime as a long representing the milliseconds since unix epoch
  * Returns: The current month (0-based).

### `PREPEND_IF_MISSING`
  * Description: Prepends the prefix to the start of the string if the string does not already start with any of the prefixes.
  * Input:
    * string - The string to be prepended.
    * prefix - The string prefix to prepend to the start of the string.
    * additionalprefix - Optional - Additional string prefix that is valid.
  * Returns: A new String if prefix was prepended, the same string otherwise.

### `PROFILE_GET`
  * Description: Retrieves a series of values from a stored profile.
  * Input:
    * profile - The name of the profile.
    * entity - The name of the entity.
    * periods - The list of profile periods to grab.  These are ProfilePeriod objects.
    * groups_list - Optional, must correspond to the 'groupBy' list used in profile creation - List (in square brackets) of groupBy values used to filter the profile. Default is the empty list, meaning groupBy was not used when creating the profile.
    * config_overrides - Optional - Map (in curly braces) of name:value pairs, each overriding the global config parameter of the same name. Default is the empty Map, meaning no overrides.
  * Returns: The selected profile measurements.

### `PROFILE_FIXED`
  * Description: The profile periods associated with a fixed lookback starting from now
  * Input:
    * durationAgo - How long ago should values be retrieved from?
    * units - The units of 'durationAgo'.
    * config_overrides - Optional - Map (in curly braces) of name:value pairs, each overriding the global config parameter of the same name. Default is the empty Map, meaning no overrides.
  * Returns: The selected profile measurement timestamps.  These are ProfilePeriod objects.

### `PROFILE_WINDOW`
  * Description: The profiler periods associated with a window selector statement from an optional reference timestamp.
  * Input:
    * windowSelector - The statement specifying the window to select.
    * now - Optional - The timestamp to use for now.
    * config_overrides - Optional - Map (in curly braces) of name:value pairs, each overriding the global config parameter of the same name. Default is the empty Map, meaning no overrides.
  * Returns: The selected profile measurement periods.  These are ProfilePeriod objects.

### `PROTOCOL_TO_NAME`
  * Description: Converts the IANA protocol number to the protocol name
  * Input:
    * IANA Number
  * Returns: The protocol name associated with the IANA number.

### `REDUCE`
  * Description: Reduces a list by a binary lambda expression. That is, the expression takes two arguments.  Usage example: `REDUCE( [ 1, 2, 3 ] , (x, y) -> x + y, 0)` would sum the input list, yielding `6`.
  * Input:                      
    * list - List of arguments.
    * binary_operation - The lambda expression function to apply to reduce the list. It is assumed that this takes two arguments, the first being the running total and the second being an item from the list.
    * initial_value - The initial value to use.
  * Returns: The reduction of the list.
  
### `REGEXP_MATCH`
  * Description: Determines whether a regex matches a string
  * Input:
    * string - The string to test
    * pattern - The proposed regex pattern
  * Returns: True if the regex pattern matches the string and false if otherwise.
  
### `REGEXP_GROUP_VAL`
  * Description: Returns the value of a group in a regex against a string
  * Input:
    * string - The string to test
    * pattern - The proposed regex pattern
    * group - The integer that selects what group to select, starting at 1
  * Returns: The value of the group, or null if not matched or no group at index.

### `STRING_ENTROPY`
  * Description: Computes the base-2 shannon entropy of a string.
  * Input:
    * input - String 
  * Returns: The base-2 shannon entropy of the string (https://en.wikipedia.org/wiki/Entropy_(information_theory)#Definition).  The unit of this is bits.

### `SPLIT`
  * Description: Splits the string by the delimiter.
  * Input:
    * input - String to split
    * delim - String delimiter
  * Returns: List of strings

### `STARTS_WITH`
  * Description: Determines whether a string starts with a prefix
  * Input:
    * string - The string to test
    * prefix - The proposed prefix
  * Returns: True if the string starts with the specified prefix and false if otherwise

### `SYSTEM_ENV_GET`
  * Description: Returns the value associated with an environment variable
  * Input:
    * env_var - Environment variable name to get the value for
  * Returns: String

### `SYSTEM_PROPERTY_GET`
  * Description: Returns the value associated with a Java system property
  * Input:
    * key - Property to get the value for
  * Returns: String

### `TO_DOUBLE`
  * Description: Transforms the first argument to a double precision number
  * Input:
    * input - Object of string or numeric type
  * Returns: Double version of the first argument

### `TO_EPOCH_TIMESTAMP`
  * Description: Returns the epoch timestamp of the dateTime in the specified format. If the format does not have a timestamp and you wish to assume a given timestamp, you may specify the timezone optionally.
  * Input:
    * dateTime - DateTime in String format
    * format - DateTime format as a String
    * timezone - Optional timezone in String format
  * Returns: Epoch timestamp
  
### `TO_FOAT`
  * Description: Transforms the first argument to a float
  * Input:
    * input - Object of string or numeric type
  * Returns: Float version of the first argument

### `TO_INTEGER`
  * Description: Transforms the first argument to an integer
  * Input:
    * input - Object of string or numeric type
  * Returns: Integer version of the first argument

### `TO_LONG`
  * Description: Transforms the first argument to a long integer
  * Input:
    * input - Object of string or numeric type
  * Returns: Long version of the first argument

### `TO_LOWER`
  * Description: Transforms the first argument to a lowercase string
  * Input:
    * input - String
  * Returns: Lowercase string

### `TO_STRING`
  * Description: Transforms the first argument to a string
  * Input:
    * input - Object
  * Returns: String

### `TO_UPPER`
  * Description: Transforms the first argument to an uppercase string
  * Input:
    * input - String
  * Returns: Uppercase string

### `TRIM`
  * Description: Trims whitespace from both sides of a string.
  * Input:
    * input - String
  * Returns: String

### `URL_TO_HOST`
  * Description: Extract the hostname from a URL.
  * Input:
    * url - URL in String form
  * Returns: The hostname from the URL as a String.  e.g. URL_TO_HOST('http://www.yahoo.com/foo') would yield 'www.yahoo.com'

### `URL_TO_PATH`
  * Description: Extract the path from a URL.
  * Input:
    * url - URL in String form
  * Returns: The path from the URL as a String.  e.g. URL_TO_PATH('http://www.yahoo.com/foo') would yield 'foo'

### `URL_TO_PORT`
  * Description: Extract the port from a URL.  If the port is not explicitly stated in the URL, then an implicit port is inferred based on the protocol.
  * Input:
    * url - URL in string form
  * Returns: The port used in the URL as an integer (for example, URL_TO_PORT('http://www.yahoo.com/foo') would yield 80)

### `URL_TO_PROTOCOL`
  * Description: Extract the protocol from a URL.
  * Input:
    * url - URL in String form
  * Returns: The protocol from the URL as a String. e.g. URL_TO_PROTOCOL('http://www.yahoo.com/foo') would yield 'http'

### `WEEK_OF_MONTH`
  * Description: The numbered week within the month.  The first week within the month has a value of 1.
  * Input:
    * dateTime - The datetime as a long representing the milliseconds since unix epoch
  * Returns: The numbered week within the month.

### `WEEK_OF_YEAR`
  * Description: The numbered week within the year.  The first week in the year has a value of 1.
  * Input:
    * dateTime - The datetime as a long representing the milliseconds since unix epoch
  * Returns: The numbered week within the year.

### `YEAR`
  * Description: The number representing the year. 
  * Input:
    * dateTime - The datetime as a long representing the milliseconds since unix epoch
  * Returns: The current year

### `ZIP`
  * Description: Zips lists into a single list where the ith element is an list containing the ith items from the constituent lists.
  See [python](https://docs.python.org/3/library/functions.html#zip)
  and [wikipedia](https://en.wikipedia.org/wiki/Convolution_(computer_science)) for more context.
  * Input:
    * list* - Lists to zip.
  * Returns: The zip of the lists.  The returned list is the min size of all the lists. e.g. `ZIP( [ 1, 2 ], [ 3, 4, 5] ) == [ [1, 3], [2, 4] ]`

### `ZIP_LONGEST`
  * Description: Zips lists into a single list where the ith element is an list containing the ith items from the constituent lists.
  See [python](https://docs.python.org/3/library/itertools.html#itertools.zip_longest)
  and [wikipedia](https://en.wikipedia.org/wiki/Convolution_(computer_science)) for more context.
  * Input:
    * list* - Lists to zip.
  * Returns: The zip of the lists.  The returned list is the max size of all the lists.  Empty elements are null e.g. `ZIP_LONGEST( [ 1, 2 ], [ 3, 4, 5] ) == [ [1, 3], [2, 4], [null, 5] ]`

The following is an example query (i.e. a function which returns a
boolean) which would be seen possibly in threat triage:

`IN_SUBNET( ip, '192.168.0.0/24') or ip in [ '10.0.0.1', '10.0.0.2' ] or exists(is_local)`

This evaluates to true precisely when one of the following is true:
* The value of the `ip` field is in the `192.168.0.0/24` subnet
* The value of the `ip` field is `10.0.0.1` or `10.0.0.2`
* The field `is_local` exists

The following is an example transformation which might be seen in a
field transformation:

`TO_EPOCH_TIMESTAMP(timestamp, 'yyyy-MM-dd HH:mm:ss', MAP_GET(dc, dc2tz, 'UTC'))`

For a message with a `timestamp` and `dc` field, we want to set the
transform the timestamp to an epoch timestamp given a timezone which we
will lookup in a separate map, called `dc2tz`.

This will convert the timestamp field to an epoch timestamp based on the 
* Format `yyyy-MM-dd HH:mm:ss`
* The value in `dc2tz` associated with the value associated with field
  `dc`, defaulting to `UTC`

## Stellar Benchmarks

A microbenchmarking utility is included to assist in executing microbenchmarks for Stellar functions.
The utility can be executed via maven using the `exec` plugin, like so, from the `metron-common` directory:

```
mvn -DskipTests clean package && \
mvn exec:java -Dexec.mainClass="org.apache.metron.stellar.common.benchmark.StellarMicrobenchmark" -Dexec.args="..."
 ```
where `exec.args` can be one of the following:
```
    -e,--expressions <FILE>   Stellar expressions
    -h,--help                 Generate Help screen
    -n,--num_times <NUM>      Number of times to run per expression (after
                              warmup). Default: 1000
    -o,--output <FILE>        File to write output.
    -p,--percentiles <NUM>    Percentiles to calculate per run. Default:
                              50.0,75.0,95.0,99.0
    -v,--variables <FILE>     File containing a JSON Map of variables to use
    -w,--warmup <NUM>         Number of times for warmup per expression.
                              Default: 100
```

For instance, to run with a set of Stellar expression in file `/tmp/expressions.txt`:
```
 # simple functions
 TO_UPPER('casey')
 TO_LOWER(name)
 # math functions
 1 + 2*(3 + int_num) / 10.0
 1.5 + 2*(3 + double_num) / 10.0
 # conditionals
 if ('foo' in ['foo']) OR one == very_nearly_one then 'one' else 'two'
 1 + 2*(3 + int_num) / 10.0
 #Network funcs
 DOMAIN_TO_TLD(domain)
 DOMAIN_REMOVE_SUBDOMAINS(domain)
```
And variables in file `/tmp/variables.json`:
```
{
  "name" : "casey",
  "int_num" : 1,
  "double_num" : 17.5,
  "one" : 1,
  "very_nearly_one" : 1.000001,
  "domain" : "www.google.com"
}
```

Written to file `/tmp/output.txt` would be the following command:
```
mvn -DskipTests clean package && \
mvn exec:java -Dexec.mainClass="org.apache.metron.stellar.common.benchmark.StellarMicrobenchmark" \
-Dexec.args="-e /tmp/expressions.txt -v /tmp/variables.json -o ./output.json"
 ```

## Stellar Shell

The Stellar Shell is a REPL (Read Eval Print Loop) for the Stellar language that helps in debugging, troubleshooting, and learning Stellar.  It can also be used as a language-checking resource while interacting with a live Metron cluster.

The Stellar DSL (domain specific language) is used to act upon streaming data within Apache Storm.  It is difficult to troubleshoot Stellar when it can only be executed within a Storm topology.  This REPL is intended to help mitigate that problem by allowing a user to replicate behavior encountered in production, isolate initialization errors, or understand function resolution problems.  Because it can be run from the command line on any node with Metron installed, it can help the user understand environmental problems that may be interfering with Stellar running in Storm servers.

The shell supports customization via `~/.inputrc` as it is
backed by a proper readline implementation.  

Shell-like operations are supported such as 
* reverse search via ctrl-r
* autocomplete of Stellar functions and variables via tab
  * NOTE: Stellar functions are read via a classpath search which
    happens in the background.  Until that happens, autocomplete will not include function names. 
* emacs or vi keybindings for edit mode

Note: Stellar classpath configuration from the global config is honored here if the REPL knows about zookeeper.

### Getting Started

To run the Stellar Shell from within a deployed Metron cluster, run the following command on the host where Metron is installed.
```
$ $METRON_HOME/bin/stellar

Stellar, Go!
{es.clustername=metron, es.ip=node1, es.port=9300, es.date.format=yyyy.MM.dd.HH}

[Stellar]>>> %functions
BLOOM_ADD, BLOOM_EXISTS, BLOOM_INIT, BLOOM_MERGE, DAY_OF_MONTH, DAY_OF_WEEK, DAY_OF_YEAR, ...

[Stellar]>>> ?PROTOCOL_TO_NAME
PROTOCOL_TO_NAME
 desc: Convert the IANA protocol number to the protocol name       
 args: IANA Number                                                 
  ret: The protocol name associated with the IANA number.          

[Stellar]>>> ip.protocol := 6
6
[Stellar]>>> PROTOCOL_TO_NAME(ip.protocol)
TCP
```

### Command Line Options

```
$ $METRON_HOME/bin/stellar -h
usage: stellar
 -h,--help              Print help
 -irc,--inputrc <arg>   File containing the inputrc if not the default
                        ~/.inputrc
 -v,--variables <arg>   File containing a JSON Map of variables
 -z,--zookeeper <arg>   Zookeeper URL fragment in the form [HOSTNAME|IPADDRESS]:PORT
 -na,--no_ansi          Make the input prompt not use ANSI colors.
```

#### `-v, --variables`
*Optional*

Optionally load a JSON map which contains variable assignments.  This is
intended to give you the ability to save off a message from Metron and
work on it via the REPL.

#### `-z, --zookeeper`

*Optional*

Attempts to connect to Zookeeper and read the Metron global configuration.  Stellar functions may require the global configuration to work properly.  If found, the global configuration values are printed to the console.
If specified, then the classpath may be augmented by the paths specified in the stellar config in the global config.

```
$ $METRON_HOME/bin/stellar -z node1:2181
Stellar, Go!
{es.clustername=metron, es.ip=node1, es.port=9300, es.date.format=yyyy.MM.dd.HH}
[Stellar]>>> 
```

### Variable Assignment

Stellar has no concept of variable assignment.  For testing and
debugging purposes, it is important to be able to create variables that
simulate data contained within incoming messages.  The REPL has created
a means for a user to perform variable assignment outside of the core
Stellar language.  This is done via the `:=` operator, such as 
`foo := 1 + 1` would assign the result of the stellar expression `1 + 1` to the
variable `foo`.

```
[Stellar]>>> foo := 2 + 2
4.0
[Stellar]>>> 2 + 2
4.0
```

### Magic Commands

The REPL has a set of magic commands that provide the REPL user with information about the Stellar execution environment.  The following magic commands are supported.

#### `%functions`

This command lists all functions resolvable in the Stellar environment.  Stellar searches the classpath for Stellar functions.  This can make it difficult in some cases to understand which functions are resolvable.  

```
[Stellar]>>> %functions
BLOOM_ADD, BLOOM_EXISTS, BLOOM_INIT, BLOOM_MERGE, DAY_OF_MONTH, DAY_OF_WEEK, DAY_OF_YEAR, 
DOMAIN_REMOVE_SUBDOMAINS, DOMAIN_REMOVE_TLD, DOMAIN_TO_TLD, ENDS_WITH, GET, GET_FIRST, 
GET_LAST, IN_SUBNET, IS_DATE, IS_DOMAIN, IS_EMAIL, IS_EMPTY, IS_INTEGER, IS_IP, IS_URL, 
JOIN, LENGTH, MAAS_GET_ENDPOINT, MAAS_MODEL_APPLY, MAP_EXISTS, MAP_GET, MONTH, PROTOCOL_TO_NAME, 
REGEXP_MATCH, SPLIT, STARTS_WITH, STATS_ADD, STATS_COUNT, STATS_GEOMETRIC_MEAN, STATS_INIT, 
STATS_KURTOSIS, STATS_MAX, STATS_MEAN, STATS_MERGE, STATS_MIN, STATS_PERCENTILE, 
STATS_POPULATION_VARIANCE, STATS_QUADRATIC_MEAN, STATS_SD, STATS_SKEWNESS, STATS_SUM, 
STATS_SUM_LOGS, STATS_SUM_SQUARES, STATS_VARIANCE, TO_DOUBLE, TO_EPOCH_TIMESTAMP, TO_FLOAT, 
TO_INTEGER, TO_LOWER, TO_STRING, TO_UPPER, TRIM, URL_TO_HOST, URL_TO_PATH, URL_TO_PORT, 
URL_TO_PROTOCOL, WEEK_OF_MONTH, WEEK_OF_YEAR, YEAR
[Stellar]>>> 
```

#### `%vars` 

Lists all variables in the Stellar environment.

```
Stellar, Go!
{es.clustername=metron, es.ip=node1, es.port=9300, es.date.format=yyyy.MM.dd.HH}
[Stellar]>>> %vars
[Stellar]>>> foo := 2 + 2
4.0
[Stellar]>>> %vars
foo = 4.0
```

#### `?<function>`

Returns formatted documentation of the Stellar function.  Provides the description of the function along with the expected arguments.

```
[Stellar]>>> ?BLOOM_ADD
BLOOM_ADD
 desc: Adds an element to the bloom filter passed in               
 args: bloom - The bloom filter, value* - The values to add        
  ret: Bloom Filter                                                
[Stellar]>>> ?IS_EMAIL
IS_EMAIL
 desc: Tests if a string is a valid email address                  
 args: address - The String to test                                
  ret: True if the string is a valid email address and false otherwise.
[Stellar]>>> 
```

### Advanced Usage

To run the Stellar Shell directly from the Metron source code, run a command like the following.  Ensure that Metron has already been built and installed with `mvn clean install -DskipTests`.
```
$ mvn exec:java \
   -Dexec.mainClass="org.apache.metron.stellar.common.shell.StellarShell" \
   -pl metron-platform/metron-enrichment
...
Stellar, Go!
Please note that functions are loading lazily in the background and will be unavailable until loaded fully.
[Stellar]>>> Functions loaded, you may refer to functions now...
[Stellar]>>> %functions
ABS, APPEND_IF_MISSING, BIN, BLOOM_ADD, BLOOM_EXISTS, BLOOM_INIT, BLOOM_MERGE, CHOMP, CHOP, COUNT_MATCHES, DAY_OF_MONTH, DAY_OF_WEEK, DAY_OF_YEAR, DOMAIN_REMOVE_SUBDOMAINS, DOMAIN_REMOVE_TLD, DOMAIN_TO_TLD, ENDS_WITH, ENRICHMENT_EXISTS, ENRICHMENT_GET, FILL_LEFT, FILL_RIGHT, FILTER, FORMAT, GEO_GET, GET, GET_FIRST, GET_LAST, HLLP_ADD, HLLP_CARDINALITY, HLLP_INIT, HLLP_MERGE, IN_SUBNET, IS_DATE, IS_DOMAIN, IS_EMAIL, IS_EMPTY, IS_INTEGER, IS_IP, IS_URL, JOIN, LENGTH, LIST_ADD, MAAS_GET_ENDPOINT, MAAS_MODEL_APPLY, MAP, MAP_EXISTS, MAP_GET, MONTH, OUTLIER_MAD_ADD, OUTLIER_MAD_SCORE, OUTLIER_MAD_STATE_MERGE, PREPEND_IF_MISSING, PROFILE_FIXED, PROFILE_GET, PROFILE_WINDOW, PROTOCOL_TO_NAME, REDUCE, REGEXP_MATCH, SPLIT, STARTS_WITH, STATS_ADD, STATS_BIN, STATS_COUNT, STATS_GEOMETRIC_MEAN, STATS_INIT, STATS_KURTOSIS, STATS_MAX, STATS_MEAN, STATS_MERGE, STATS_MIN, STATS_PERCENTILE, STATS_POPULATION_VARIANCE, STATS_QUADRATIC_MEAN, STATS_SD, STATS_SKEWNESS, STATS_SUM, STATS_SUM_LOGS, STATS_SUM_SQUARES, STATS_VARIANCE, STRING_ENTROPY, SYSTEM_ENV_GET, SYSTEM_PROPERTY_GET, TO_DOUBLE, TO_EPOCH_TIMESTAMP, TO_FLOAT, TO_INTEGER, TO_LONG, TO_LOWER, TO_STRING, TO_UPPER, TRIM, URL_TO_HOST, URL_TO_PATH, URL_TO_PORT, URL_TO_PROTOCOL, WEEK_OF_MONTH, WEEK_OF_YEAR, YEAR
```

Changing the project passed to the `-pl` argument will define which dependencies are included and ultimately which Stellar functions are available within the shell environment.  

This can be useful for troubleshooting function resolution problems.  The previous example defines which functions are available during Enrichment.  For example, to determine which functions are available within the Profiler run the following.

```
 $ mvn exec:java \
   -Dexec.mainClass="org.apache.metron.stellar.common.shell.StellarShell" \
   -pl metron-analytics/metron-profiler
...
Stellar, Go!
Please note that functions are loading lazily in the background and will be unavailable until loaded fully.
[Stellar]>>> Functions loaded, you may refer to functions now...
%functions
ABS, APPEND_IF_MISSING, BIN, BLOOM_ADD, BLOOM_EXISTS, BLOOM_INIT, BLOOM_MERGE, CHOMP, CHOP, COUNT_MATCHES, DAY_OF_MONTH, DAY_OF_WEEK, DAY_OF_YEAR, DOMAIN_REMOVE_SUBDOMAINS, DOMAIN_REMOVE_TLD, DOMAIN_TO_TLD, ENDS_WITH, FILL_LEFT, FILL_RIGHT, FILTER, FORMAT, GET, GET_FIRST, GET_LAST, HLLP_ADD, HLLP_CARDINALITY, HLLP_INIT, HLLP_MERGE, IN_SUBNET, IS_DATE, IS_DOMAIN, IS_EMAIL, IS_EMPTY, IS_INTEGER, IS_IP, IS_URL, JOIN, LENGTH, LIST_ADD, MAAS_GET_ENDPOINT, MAAS_MODEL_APPLY, MAP, MAP_EXISTS, MAP_GET, MONTH, OUTLIER_MAD_ADD, OUTLIER_MAD_SCORE, OUTLIER_MAD_STATE_MERGE, PREPEND_IF_MISSING, PROFILE_FIXED, PROFILE_GET, PROFILE_WINDOW, PROTOCOL_TO_NAME, REDUCE, REGEXP_MATCH, SPLIT, STARTS_WITH, STATS_ADD, STATS_BIN, STATS_COUNT, STATS_GEOMETRIC_MEAN, STATS_INIT, STATS_KURTOSIS, STATS_MAX, STATS_MEAN, STATS_MERGE, STATS_MIN, STATS_PERCENTILE, STATS_POPULATION_VARIANCE, STATS_QUADRATIC_MEAN, STATS_SD, STATS_SKEWNESS, STATS_SUM, STATS_SUM_LOGS, STATS_SUM_SQUARES, STATS_VARIANCE, STRING_ENTROPY, SYSTEM_ENV_GET, SYSTEM_PROPERTY_GET, TO_DOUBLE, TO_EPOCH_TIMESTAMP, TO_FLOAT, TO_INTEGER, TO_LONG, TO_LOWER, TO_STRING, TO_UPPER, TRIM, URL_TO_HOST, URL_TO_PATH, URL_TO_PORT, URL_TO_PROTOCOL, WEEK_OF_MONTH, WEEK_OF_YEAR, YEAR 
```

# Stellar Configuration

Stellar can be configured in a variety of ways from the [Global Configuration](../../metron-platform/metron-common/README.md#global-configuration).
In particular, there are three main configuration parameters around configuring Stellar:
* `stellar.function.paths`
* `stellar.function.resolver.includes`
* `stellar.function.resolver.excludes`

## `stellar.function.paths`

If specified, Stellar will use a custom classloader which will wrap the
context classloader and allow for the resolution of classes stored in jars
not shipped with Metron and stored in a variety of mediums:
* On HDFS
* In tar.gz files
* At http/s locations
* At ftp locations

This path is a comma separated list of 
* URIs
* URIs with a regex pattern ending it for matching within a directory

```json
{
 ...
  "stellar.function.paths" : "hdfs://node1:8020/apps/metron/stellar/metron-management-0.4.1.jar, hdfs://node1:8020/apps/metron/3rdparty/.*.jar"
}
```

Please be aware that this classloader does not reload functions dynamically
and the classpath specified here in the global config is read on topology start.
  A change in classpath, to be picked up, would necessitate a topology restart
at the moment

## `stellar.function.resolver.{includes,excludes}`

If specified, this defines one or more regular expressions applied to the classes implementing the Stellar function
that specify what should be included when searching for Stellar functions.
* `stellar.function.resolver.includes` defines the list of classes to include.
* `stellar.function.resolver.excludes` defines the list of classes to exclude.

```json
{
 ...
  "stellar.function.resolver.includes" : "org.apache.metron.*,com.myorg.stellar.*"
}
```

