# NAME

Data::Processor - Transform Perl Data Structures, Validate Data against a Schema, Produce Data from a Schema, or produce documentation directly from information in the Schema.

# SYNOPSIS

    use Data::Processor;
    my $schema = {
      section => {
          description => 'a section with a few members',
          error_msg   => 'cannot find "section" in config',
          members => {
              foo => {
                  # value restriction either with a regex..
                  value => qr{f.*},
                  description => 'a string beginning with "f"'
              },
              bar => {
                  # ..or with a validator callback.
                  validator => sub {
                      my $self   = shift;
                      my $parent = shift;
                      # undef is "no-error" -> success.
                      no strict 'refs';
                      return undef
                          if $self->{value} == 42;
                  }
              },
              wuu => {
                  optional => 1
              }
          }
      }
    };

    my $p = Data::Processor->new($schema);

    my $data = {
      section => {
          foo => 'frobnicate',
          bar => 42,
          # "wuu" being optional, can be omitted..
      }
    };

    my $error_collection = $p->validate($data, verbose=>0);
    # no errors :-)

    # in case of errors:
    # ------------------
    # print each error on one line.
    say $error_collection;

    # same
    for my $e ($error_collection->as_array){
        say $e;
        # do more..
    }

# DESCRIPTION

Data::Processor is a tool for transforming, verifying, and producing Perl data structures from / against a schema, defined as a Perl data structure.

# METHODS

## new

    my $processor = Data::Processor->new($schema);

optional parameters:
\- indent: count of spaces to insert when printing in verbose mode. Default 4
\- depth: level at which to start. Default is 0.
\- verbose: Set to a true value to print messages during processing.

## validate
Validate the data against a schema. The schema either needs to be present
already or be passed as an argument.

    my $error_collection = $processor->validate($data, verbose=>0);

## validate\_schema

check that the schema is valid.
This method gets called upon creation of a new Data::Processor object.

    my $error_collection = $processor->validate_schema();

## merge\_schema

merges another schema into the schema (optionally at a specific node)

    my $error_collection = $processor->merge_schema($schema_2);

merging rules:
 - merging transformers will result in an error
 - merge checks if all merged elements match existing elements
 - non existing elements will be added from merging schema
 - validators from existing and merging schema get combined

## schema

Returns the schema. Useful after schema merging.

## transform\_data

Transform one key in the data according to rules specified
as callbacks that themodule calls for you.
Transforms the data in-place.

    my $validator = Data::Processor::Validator->new($schema, data => $data)
    my $error_string = $processor->transform($key, $schema_key, $value);

This is not tremendously useful at the moment, especially because validate()
transforms during validation.

## make\_data

Writes a data template using the information found in the schema.

    my $data = $processor->make_data(data=>$data);

## make\_pod

Write descriptive pod from the schema.

    my $pod_string = $processor->make_pod();

# SCHEMA REFERENCE

## Top-level keys and members

The schema is described by a nested hash. At the top level, and within a
members definition, the keys are the same as the structure you are
describing. So for example:

    my $schema = {
        coordinates => {
            members => {
                x => {
                    description => "the x coordinate",
                },
                y => {
                    description => "the y coordinate",
                },
            }
        }
    };

This schema describes a structure which might look like this:

    { coordinates => { x => 1, y => 2} }

Obviously this can be nested all the way down:

    my $schema = {
       house => {
          members => {
              bungalow => {
                  members => {
                      rooms => {
                        #...
                      }
                  }
              }
          }
       }
    };

## array

To have a key point to an array of things, simply use the array key. So:

    my $schema = {
       houses => {
          array => 1,
       }
    };

Would describe a structure like:

    { houses => [] }

And of course you can nest within here so:

    my $schema = {
       houses => {
          array => 1,
          members => {
              name => {},
              windows => {
                  array => 1,
              }
          },
       },
    };

Might describe:

    {
      houses => [
         { name => 'bob',
           windows => []},
         { name => 'harry',
           windows => []},
      ]
    }

## description

The description key within a definition describes that value:

    my $schema = {
        x => { description => 'The x coordinate' },
    };

## error\_msg

The error\_msg key can be set to provide extra context for when a value is not
found or fails the [value](#value) test.

## optional

Most values are required by default. To reverse this use the "optional" key:

    my $schema = {
        x => {
          optional => 1,
        },
        y => {
          # required
        },
    };

## regex

**Treating regular expressions as keys**

If you set "regex" within a definition then it's key will be treated as a
regular expression.

    my $schema = {
       'color_.+' => {
          regex => 1
       },
    };
    my $data = { color_red => 'red', color_blue => 'blue'};
    Data::Processor->new($schema)->validate($data);

## transformer

**transform the data for further processing**

Transformer maps to a sub ref which will be passed the value and the containing
structure. Your return value provides the new value.

    my $schema = {
       x => {
           transformer => sub{
              my( $value, $section ) = @_;
              $value = $value + 1;
              return $value;
           }
       }
    };
    my $data = { x => 1 };
    my $p = Data::Processor->new($schema);
    my $val = Data::Processor::Validator->new( $schema, data => $data);
    $p->transform_data('x', 'x', $val);
    say $data->{x}; #will print 2

If you wish to provide an error from the transformer you should die with a
hash reference with a key of "msg" mapping to your error:

    my $schema = {
       x => {
            transformer => sub{
                die { msg => "SOMETHING IS WRONG" };
            }
       },
    };

    my $p = Data::Processor->new($schema);
    my $data = { x => 1 };
    my $val = Data::Processor::Validator->new( $schema, data => $data);
    my $error = $p->transform_data('x', 'x', $val);

    say $error; # will print: error transforming 'x': SOMETHING IS WRONG

The transformer is called before any validator, so:

    my $schema = {
       x => {
           transformer => sub{
              my( $value, $section ) = @_;
              return $value + 1;
           },
           validator => sub{
              my( $value ) = @_;
              if( $value < 2 ){
                 return "too low"
              }
           },
       },
    };
    my $p = Data::Processor->new( $schema );
    my $data = { x => 1 };
    my $errors = $p->validate();
    say $errors->count; # will print 0
    say $data->{x}; # will print 2

## value

**checking against regular expression**

To check a value against a regular expression you can use the _value_ key
within a definition, mapped to a quoted regex:

    my $schema = {
        x => {
           value => qr{\d+}
        }
    };

## validator

**checking more complex values using a callback**

To conduct extensive checks you can use _validator_ and provide a
callback. Your sub will be passed the value and it's container. If you return
anything it will be regarded as an error message, so to indicate a valid
value you return nothing:

    my $schema = {
       bob => {
         validator => sub{
            my( $value, $section ) = @_;
            if( $value ne 'bob' ){
               return "Bob must equal bob!";
            }
            return;
         },
       },
    };
    my $p = Data::Processor->new($schema);
    # would validate:
    $p->validate({ bob => "bob" });
    # would fail:
    $p->validate({ bob => "harry"});

See also [Data::Processor::ValidatorFactory](https://metacpan.org/pod/Data%3A%3AProcessor%3A%3AValidatorFactory)

### Validator objects

Validator may also be an object, in this case the object must implement a
"validate" method.

The "validate" method should return undef if valid, or an error message string if there is a problem.

    package FiveChecker;

    sub new {
        bless {}, shift();
    }

    sub validate{
        my( $self, $val ) = @_;
        $val == 5 or return "I wanted five!";
        return;
    }
    package main;

    my $checker = FiveChecker->new;
    my $schema = (
        five => (
            validator => $checker,
        ),
    );
    my $dp = Data::Processor->new($schema);
    $dp->validate({five => 6}); # fails
    $dp->validate({five => 5}); # passes

You can for example use MooseX::Types and Type::Tiny type constraints that are objects
which offer validate methods which work this way.

    use Types::Standard -all;

    # ... in schema ...
         foo => {
             validator => ArrayRef[Int],
             description => 'an arrayref of integers'
         },

# AUTHOR

Matthias Bloch <matthias.bloch@puffin.ch>

# COPYRIGHT

Copyright 2015- Matthias Bloch

# LICENSE

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself.
