# Wavefront OBJ to Indexed Mesh

# Introduction

A common format for storing meshes for hobby projects and prototyping renderers is [Wavefront .obj](https://en.wikipedia.org/wiki/Wavefront_.obj_file) (referred to as OBJ for the rest of this article) due to its simplicity and ease of parsing. However, the format has a drawback: a vertex can contain indices to different positions, texture coordinates, and/or normals, as shown below.

```
v 1.000000 -1.000000 -1.000000
v 1.000000 -1.000000 1.000000
v -1.000000 -1.000000 1.000000
vt 0.748573 0.750412
vn 0.000000 0.000000 -1.000000
vn 0.000000 0.000000 1.000000
f 1/1/1 2/1/1 3/1/2
```

This is great for reducing file sizes but not great because Graphics APIs like OpenGL cannot be told to use one index for position, another for texture coordinates, and another for the normal. Therefore, we must process the OBJ file to a format we can use with the API of our choice. In this article, I want to showcase an algorithm for doing so.

> [!NOTE]
> The algorithm described can be applied to other Graphics APIs where indexed draws only use one index for all vertex attributes.

# Algorithm

The algorithm is as follows:

1. Create an associative container (dictionary/unordered_map) with the tuple <v, vt, vn> as the key and value as a corresponding newly generated index. Create an array of the tuple <v, vt, vn> for reverse mapping and an array for indices.
2. For each vertex of each face, you will have a <v, vt, vn> tuple. If that tuple already exists in the dictionary, store the associated value (index) in the indices array. Otherwise, add a new entry to the map using the tuple as the key and a new, unused index as the value. Append the tuple to the reverse mapping array and index to the indices array.
3. Loop over the reverse mapping array, appending the corresponding values from the <v, vt, vn> tuple to output positions, texture coordinates, and normals arrays.

Here is a pseudo-code adapting the above algorithm for mapping OBJ to a format that OpenGL can use.

```
struct vertex_key{ v, vt, vn }

dictionary<vertex_key, index> vertices
list<vertex_key> vertex_mapping
list<index> indices

for each vertex_key in face:
  if vertex_key exists in vertices:
    indices.append(vertices[vertex_key].index)
  else:
    index = unused_index_value()
    vertices.add(vertex_key, index)
    vertex_mapping.append(vertex_key)
    indices.append(index)

list<vec3> positions
list<vec2> tex_coords
list<vec3> normals

for each vertex_key in vertex_mapping:
  positions.append(obj_positions[vertex_key.v])
  tex_coords.append(obj_tex_coords[vertex_key.vt])
  normals.append(obj_normals[vertex_key.vn])

return positions, tex_coords, normals, indices
```

> [!TIP]
> [Euler's Characteristic](https://en.wikipedia.org/wiki/Euler_characteristic) states that large (closed) triangular meshes typically have twice as many triangles as vertices. Therefore, the starting capacity of `vertex_mapping` can be estimated as `indices / 2`.

# Example

This code snippet is an implementation of the above algorithm in C++.

```cpp
// The following are outputs from a function that reads the OBJ file.
std::vector<glm::vec3> obj_v;
std::vector<glm::vec2> obj_vt;
std::vector<glm::vec3> obj_vn;
std::vector<int> obj_v_indices;
std::vector<int> obj_vn_indices;
std::vector<int> obj_vt_indices;

struct Key {
  int v_index;
  int vt_index;
  int vn_index;
  bool operator==(Key const &other) const {
	  return v_index == other.v_index &&
	         vt_index == other.vt_index &&
	         vn_index == other.vn_index;
  }
};

struct Hasher {
  // https://stackoverflow.com/a/17017281
  size_t operator()(Key const &key) const {
    size_t result = 17;
    result = result * 31 + std::hash<s32>()(key.v_index);
    result = result * 31 + std::hash<s32>()(key.vn_index);
    result = result * 31 + std::hash<s32>()(key.vt_index);
    return result;
  }
};

std::unordered_map<Key, unsigned short, Hasher> vertices;
std::vector<Key> vertex_mapping;
std::vector<unsigned short> indices;

for (size_t i = 0; i < obj_v_indices.size(); ++i) {
  Key const key{
    .v_index = obj_v_indices[i],
    .vt_index = obj_vt_indices[i],
    .vn_index = obj_vn_indices[i],
  };
  if (auto const it = vertices.find(key); it != std::end(vertices)) {
    indices.push_back(it->second);
  } else {
    unsigned short const index = (unsigned short)vertex_mapping.size();
    vertex_mapping.push_back(key);
    vertices[key] = index;
    indices.push_back(index);
  }
}

std::vector<glm::vec3> positions;
std::vector<glm::vec2> tex_coords;
std::vector<glm::vec3> normals;

for (size_t i = 0; i < vertex_mapping.size(); ++i) {
  positions.push_back(obj_v[vertex_mapping[i].v_index]);
  tex_coords.push_back(obj_vt[vertex_mapping[i].vt_index]);
  normals.push_back(obj_vn[vertex_mapping[i].vn_index]);
}
```
