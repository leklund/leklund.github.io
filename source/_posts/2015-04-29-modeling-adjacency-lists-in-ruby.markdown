---
layout: post
title: "Modeling Adjacency Lists in Ruby"
description: 'Classes to model adjacency lists in ruby'
date: 2015-04-29 11:07:35 -0400
comments: true
published: true
categories: [ruby, lists, graphs, trees]
keywords: ruby,lists,trees,graphs
---
Normally, this would be a task I would try to keep in my data layer
since PostgreSQL is so great at doing this with recursive queries. However,
sometimes you just need to operate in Ruby and Ruby alone. Let's say I want to model
what I always call an adjacency list but what is more technically a a directed graph.
We have a set of nodes where any given node can have a parent node:

<pre>
    1
   / \
  2   3
     / \
    4   5
</pre>

We will define two classes: Tree and Node. A Node object is a very simple:

```ruby
class Node
  attr_accessor :id, :name, :parent, :children

  def initialize(opts = {})
    @id = opts[:id]
    @name = opts[:name]
    @parent = opts[:parent]

    # add children if passed an array or initialize as an empty array
    @children = opts[:children].is_a?(Array) ? opts[:children] : []

  end
end
```

We have accessors for the id which will be a unique identifier and will give us the ability to easily retrieve
nodes from the tree. The name is just an arbitrary string. Parent is just another Node object. Children is an array of
Node objects that all have the current node as a parent. The nodes above would could be modeled as:

```ruby
n1 = Node.new id: 1, name: 'one'
n2 = Node.new id: 2, parent: n1, name: 'two'
n3 = Node.new id: 3, parent: n1, name: 'three'
n4 = Node.new id: 4, parent: n3, name: 'four'
n5 = Node.new id: 5, parent: n3, name: 'five'
n1.children = [n2, n3]
n3.children = [n4, n5]
```

Adding children like this is tedious and we also don't have good way to traverse the tree. So let's introduce a tree class.

```ruby
class Tree
  attr_accessor :nodes

  def initialize
    @nodes = {}
  end

  def node(id)
    @nodes[id]
  end

  # helper for hash-like access
  def [](id)
    node(id)
  end

  def add_node(node)
    raise "#{node.class} != Node" if node.class != Node
    if !node.parent.nil?
      # check if the parent is part of the tree
      existing_parent = self.node(node.parent.id)
      if existing_parent.nil?
        raise "Unable to add node #{node.id} to the tree. It specifies a parent #{node.parent.id} That has not been added to the tree"
      end
      node.parent = existing_parent
      existing_parent.children.push(node.id).uniq
    end
    @nodes[node.id] = node
  end
end
```

This class gives us a single accessor for nodes which is just a hash of Node objects. We can retrieve
a node via a method or with hash-like syntax. The `add_node` method will add a node to the tree and it will also
update the children on a parent node. Now we can do fun stuff like this:

```ruby
nodes = [n1, n2, n3, n4, n5]
tree = Tree.new
nodes.each {|n| tree.add_node n}

tree.node(4).name         #=> 'four'
tree.node(4).parent.name  #=> 'three'
tree.node(1).children.map(&:id)     #=> [2,3]
```

Now lets add some functionality to traverse the tree to get ancestors as well as the path of a node. These
methods are part of the Tree class.

```ruby
def ancestors(node, ancestor_list = [])
  # traverse up the tree to get all parents
  # return the list if the parent of the current node is already in the list
  if node.parent && !ancestor_list.include?(node.parent)
    ancestor_list.push node.parent
    self.ancestors(node.parent, ancestor_list)
  else
    ancestor_list.empty? ? nil : ancestor_list.reverse
  end
end

def path(node)
  if node.parent.nil?
    path = node.id.to_s
  else
    path = ancestors(node).push(node).map{ |n| n.id.to_s) }.join ':'
  end
end
```

Ancestors will return an array of node objects ordered from the top of the graph down to the parent
of the given node. The path method uses the ancestor array to generate a human readable path to the node.
Given the same tree above:

```ruby
tree.ancestors(n4).map(&:name) #=> ["one", "three"]
tree.path(n4) #=> '1:3:4'
```

The path method could be used, for example, to generate URL paths from an XML taxonomy tree
or other data structure.

[Complete code gist.](https://gist.github.com/leklund/f1fb60ea820720463938)
