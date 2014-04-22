Coding styles are like, uh, noses.  Everyone has one and they all smell.  They're not dogmas but instead like war treaties: ways for people to stop fighting and get to work.  The goal of defining this coding style is to enable devs to be more productive by not having to worry about code formatting (much).

When writing code, aim for (1) clarity; (2) brevity, except where it conflicts with (1).  This coding style intends to help achieve that.

Headers are named like `my_header.h`.  Implementations are named like `my_implementation.cc`.

rr uses tab characters indented with 8-character offset. The code below renders with only a 4-character offset in github's HTML; view markdown source for the actual code.
```C++
/* -*- Mode: C++; tab-width: 8; c-basic-offset: 8; indent-tabs-mode: t; -*- */

/**
 * "API" comments look like this.
 */
static void local_bare_function(Type* t)
{
	// "Inline" comments look like this.
	// Note that "*" snuggles to type, |Type*|.  Not |Type *t|.
}

/**
 * Summary comments for classes encouraged.
 */
class Type {
public:
	Type(int x, string y)
		: field1(x)
		, field2(y)

	/**
	 * Nontrivial methods must have comments.
	 */
	void method();

	const std::string& obvious_inline_method() {
		// Obvious methods can elide API comments.
		return name;
	}

	int super_trivial_getter() { return time; }

private:
	// Comments describing nontrivial fields, or nontrivial relations between
	// fields, are encouraged.
	int field1;
	int field2;
};

void
Type::method()
{
	// Out-of-line definition.
}

/**
 * Trivial "data structs" can be defined as the following, to emphasize that they're
 * C-style structs.
 */
struct simple_data_container {
	int f;
	pid_t pid;
};

/**
 * "Adjective", or "attribute", types look like this.
 */
template<typename T>
class my_special_ptr {
	//...
};

// In the foo.cc file, your header is included first.
#include "foo.h"

// Followed by C system headers.  In alphabetical order.
#include <stdio.h>
#include <sys/types.h>

// Followed by C++ system headers.  In alphabetical order.
#include <string>
#include <vector>

// Followed by local headers.  In alphabetical order.
#include "config.h
#include "log.h"
// Did I mention alphabetical order? ;)

// Don't infect .cc files with |std::|s.  However, don't add toplevel |using| directives to headers.
using namespace std;

// Use C++ smart pointers when ownership semantics is important or not immediately obvious.
shared_ptr<T> bar;
unique_ptr<T> baz;

// Comments like

	// Get x from t.
	int x = t->get_x();

// don't help anyone and just occupy space.  Don't write them!

// Strive to write code clear enough that it documents itself.  When that's not possible,
// write high-level comments that define a specification and/or set of invariants for the
// implementation that follows.  Then document non-obvious parts of the implementation inline.

/**
 * Return the reciprocal of |x|, which must be non-zero.
 */
static float recip(float x)
{
	// This looks weird, but it turns out that on the VAX the code gcc generates for
	// float reciprocal ... [etc.]
	return newtons_method(x);
}
```