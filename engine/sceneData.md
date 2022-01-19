# 引擎架构之场景数据

## 整体架构
场景数据主要分为三大部分，几何数据、材质数据，以及把两者联系起来的场景数据：
1. 几何数据，主要存储顶点、顶点索引等几何信息，以 MeshData 作为主要结构类型；
2. 材质数据，主要存储材质的描述信息，以 PBR 渲染需要的属性为主，包括数值和贴图；
3. 场景数据，主要以 Node 节点作为最小结构，节点有父子兄弟关系，节点可以关联几何数据和材质数据，也可以不关联，节点的 transfrom 变化，会影响所有的子节点，其内涵是用一种树形父子关系图来描述整个场景中的对象；
下面我们先从场景数据 Scene 开始分析

## 场景数据 Scene
场景数据的作用，是用来描述几何体之间的关系，这里采用 Node 节点作为最基础的描述单位，Node 之间有父子兄弟的联系，Node 可以关联几何数据和材质数据，也可以不关联，只是作为共同的父级，描述几何变换，下面开始具体分析。

我们首先来看看，传统的父子对象结构的场景图是什么样的，如下：

        struct SceneNode {
            SceneNode* parent_;
            vector<SceneNode*> children_;
            mat4 localTransform_;
            mat4 globalTransform_;
            Mesh* mesh_;
            Material* material_;
            void Render();
        };
这个结构描述了一个节点基本的作用：
1. 描述继承关系，包括父子兄弟；
2. 关联变换矩阵；
3. 关联几何数据；
4. 关联材质数据；
5. Render 方法，根据以上数据来绘制自身；

这种面向父子关系的树形对象结构，符合人们的第一直觉，只需要把每层的对象通过父子兄弟串联起来，在通过深度遍历算法来触发每个个体的 Render 方法，就能将整个场景绘制完毕，但是这种方法在使用简单的同时，也带来了很多设计上的缺陷，如：
1. 扩展性差，面对新的场景时，不得不扩展 SceneNode，在节点上附加更多的属性；
2. 性能差，遍历节点的数据并不放在一起，导致遍历的时候，会由于局部性原理浪费性能；
3. 采用指针管理数据，容易导致循环引用、野指针等各种问题；
4. 遍历绘制的方法，使得调试过程也困难重重；

这里我们采用面向数据的设计方式，主要包括以下几点：
1. 节点不再包含数据，所有的数据以数组的形式存放在 Scene 场景对象中；
2. 节点不再是一个 Node 实体，而是一个索引，这个索引作用于所有存放数据的数组；
3. 通过索引代替指针，管理几何数据和材质数据；

        struct Scene
        {
        	// 1. 变换矩阵
            // local transformations for each node and global transforms
        	// + an array of 'dirty/changed' local transforms
        	std::vector<mat4> localTransform_;
        	std::vector<mat4> globalTransform_;

        	// list of nodes whose global transform must be recalculated
        	std::vector<int> changedAtThisFrame_[MAX_NODE_LEVEL];

            // 2. 继承关系
        	// Hierarchy component
        	std::vector<Hierarchy> hierarchy_;

            // 3. 关联几何数据
        	// Mesh component: Which node corresponds to which node
        	std::unordered_map<uint32_t, uint32_t> meshes_;

            // 4. 关联材质数据
        	// Material component: Which material belongs to which node
        	std::unordered_map<uint32_t, uint32_t> materialForNode_;

            // 5. 调试信息，包括节点名称、材质名称
        	// Node name component: Which name is assigned to the node
        	std::unordered_map<uint32_t, uint32_t> nameForNode_;

        	// List of scene node names
        	std::vector<std::string> names_;

        	// Debug list of material names
        	std::vector<std::string> materialNames_;
        };

        struct Hierarchy
        {
        	// parent for this node (or -1 for root)
        	int parent_;
        	// first child for a node (or -1)
        	int firstChild_;
        	// next sibling for a node (or -1)
        	int nextSibling_;
        	// last added node (or -1)
        	int lastSibling_;
        	// cached node level
        	int level_;
        };

## 几何数据 Mesh
几何数据的核心是如何组织顶点数据和索引数据，对于数据的组织，最重要的要求就是要紧凑，最好可以直接就给 GPU 使用，所以要在内存中紧凑排列，这里我们想到了用两个大的数组来分别存储顶点数据和索引数据，然后因为数据中的元素都是一样的类型，为了搞清楚那一部分数据属于哪一个 Mesh，我们还需要定义一个统一的描述结构，而且因为数据是在数组中存放的，所以描述结构中对于数据的引用，自然而然就可以使用索引，避免使用指针，下面来具体看下数据结构：
1. MeshData 作为内存中存储数据的代表，按照刚才说的方式保存了顶点、索引数组，以及 Mesh 几何体描述数组，和包围盒数据：

        struct MeshData
        {
        	std::vector<uint32_t> indexData_;
        	std::vector<float> vertexData_;
        	std::vector<Mesh> meshes_;
        	std::vector<BoundingBox> boxes_;
        };
2. MeshFileHeader 作为文件中的总描述符，主要用来指导文件中的数据和内存中的数据是一种什么样的对应关系，在文件中偏移的位置和对应的 MeshData 中的属性：

        struct MeshFileHeader
        {
        	// 1. 魔数，用来确定该文件是 Mesh 数据文件
            /* Unique 64-bit value to check integrity of the file */
        	uint32_t magicValue;

            // 2. Mesh 的数量，用来确定有多少个 Mesh 描述符
        	/* Number of mesh descriptors following this header */
        	uint32_t meshCount;

            // 3. 数据段开始的 offset
        	/* The offset to combined mesh data (this is the base from which the offsets in individual meshes start) */
        	uint32_t dataBlockStartOffset;

            // 4. 索引数据段的大小
        	/* How much space index data takes */
        	uint32_t indexDataSize;

            // 5. 顶点数据段的大小
        	/* How much space vertex data takes */
        	uint32_t vertexDataSize;

        	/* According to your needs, you may add additional metadata fields */
        };
3. Mesh 描述如何对应顶点和索引数据

        struct Mesh final
        {
        	// 1. 总共有多少个 LOD 层级
            /* Number of LODs in this mesh. Strictly less than MAX_LODS, last LOD offset is used as a marker only */
        	uint32_t lodCount = 1;

            // 2. 每种顶点格式的顶点构成一种 Stream，总共有多少个 Stream
        	/* Number of vertex data streams */
        	uint32_t streamCount = 0;

            // 3. 该 Mesh 对应的索引和顶点的 offset，这里是以顶点为单位，不是以字节为单位
        	/* The total count of all previous vertices in this mesh file */
        	uint32_t indexOffset = 0;

        	uint32_t vertexOffset = 0;

            // 4. 总共有多少顶点
        	/* Vertex count (for all LODs) */
        	uint32_t vertexCount = 0;

            // 5. 每种 LOD 对应的索引的起始位置
        	/* Offsets to LOD data. Last offset is used as a marker to calculate the size */
        	uint32_t lodOffset[kMaxLODs] = { 0 };

        	inline uint32_t getLODIndicesCount(uint32_t lod) const { return lodOffset[lod + 1] - lodOffset[lod]; }

            // 6. 不同 steam 对应的 offset
        	/* All the data "pointers" for all the streams */
        	uint32_t streamOffset[kMaxStreams] = { 0 };
            
            // 7. 每种 steam 中，顶点的大小，比如：一个顶点包含位置、法线、UV坐标，那就是 3 + 3 + 2 = 8
        	/* Information about stream element (size pretty much defines everything else, the "semantics" is defined by the shader) */
        	uint32_t streamElementSize[kMaxStreams] = { 0 };

        	/* We could have included the streamStride[] array here to allow interleaved storage of attributes.
         	   For this book we assume tightly-packed (non-interleaved) vertex attribute streams */

        	/* Additional information, like mesh name, can be added here */
        };

## 材质数据

        enum MaterialFlags
        {
        	sMaterialFlags_CastShadow = 0x1,
        	sMaterialFlags_ReceiveShadow = 0x2,
        	sMaterialFlags_Transparent = 0x4,
        };

        constexpr const uint64_t INVALID_TEXTURE = 0xFFFFFFFF;

		// PACKED_STRUCT 表示按照一个字节紧凑对齐
		// 这里的材质属性，和 glTF2 标准对齐
        struct PACKED_STRUCT MaterialDescription final
        {
			gpuvec4 emissiveColor_ = { 0.0f, 0.0f, 0.0f, 0.0f};
        	gpuvec4 albedoColor_   = { 1.0f, 1.0f, 1.0f, 1.0f };
        	// UV anisotropic roughness (isotropic lighting models use only the first value). ZW values are ignored
        	gpuvec4 roughness_     = { 1.0f, 1.0f, 0.0f, 0.0f }; 
        	float transparencyFactor_ = 1.0f;
        	float alphaTest_          = 0.0f;
        	float metallicFactor_     = 0.0f;
        	uint32_t flags_ = sMaterialFlags_CastShadow | sMaterialFlags_ReceiveShadow;
        	// maps
        	uint64_t ambientOcclusionMap_  = INVALID_TEXTURE;
        	uint64_t emissiveMap_          = INVALID_TEXTURE;
        	uint64_t albedoMap_            = INVALID_TEXTURE;
        	/// Occlusion (R), Roughness (G), Metallic (B) https://github.com/KhronosGroup/glTF/issues/857
        	uint64_t metallicRoughnessMap_ = INVALID_TEXTURE;
        	uint64_t normalMap_            = INVALID_TEXTURE;
        	uint64_t opacityMap_           = INVALID_TEXTURE;
        };

		struct PACKED_STRUCT gpuvec4
		{
			float x, y, z, w;

			gpuvec4() = default;
			explicit gpuvec4(float v): x(v), y(v), z(v), w(v) {}
			gpuvec4(float a, float b, float c, float d): x(a), y(b), z(c), w(d) {}
			explicit gpuvec4(const vec4& v): x(v.x), y(v.y), z(v.z), w(v.w) {}
		};

		struct PACKED_STRUCT gpumat4
		{
			float data_[16];

			gpumat4() = default;
			explicit gpumat4(const glm::mat4& m)  { memcpy(data_, glm::value_ptr(m), 16 * sizeof(float)); }
		};

## 实现对于 assimp 加载场景的转换

        int main()
        {
        	fs::create_directory("data/out_textures");

        	const auto configs = readConfigFile("data/sceneconverter.json");

        	for (const auto& cfg: configs)
        		processScene(cfg);

        	// Final step: optimize bistro scene
        	mergeBistro();

        	return 0;
        }
这里的 `configs` 用来保存需要进行转换的模型的一些信息，如下：

        [
        	{
        		"input_scene": "deps/src/bistro/Exterior/exterior.obj",
        		"output_mesh": "data/meshes/test.meshes",
        		"output_scene": "data/meshes/test.scene",
        		"output_materials": "data/meshes/test.materials",
        		"scale": 0.01,
        		"calculate_LODs": false,
        		"merge_instances": true
        	},
        	{
        		"input_scene": "deps/src/bistro/Interior/interior.obj",
        		"output_mesh": "data/meshes/test2.meshes",
        		"output_scene": "data/meshes/test2.scene",
        		"output_materials": "data/meshes/test2.materials",
        		"scale": 0.01,
        		"calculate_LODs": false,
        		"merge_instances": true
        	},
        	{
        		"input_scene": "data/meshes/orrery/scene.gltf",
        		"output_mesh": "data/meshes/test_graph.meshes",
        		"output_scene": "data/meshes/test_graph.scene",
        		"output_materials": "data/meshes/test_graph.materials",
        		"scale": 1.0,
        		"calculate_LODs": false,
        		"merge_instances": false
        	},
        	{
        		"input_scene":  "data/rubber_duck/scene.gltf",
        		"output_mesh":  "data/meshes/test_duck.meshes",
        		"output_scene": "data/meshes/test_duck.scene",
        		"output_materials": "data/meshes/test_duck.materials",
        		"scale": 1.0,
        		"calculate_LODs": false,
        		"merge_instances": false
        	}
        ]
`mergeBistro()` 函数主要用来合并场景数据，后面再详细分析，下面先主要来看`processScene()` 函数。

### processScene 转换模型数据

        MeshData g_MeshData;
        uint32_t g_indexOffset = 0;
        uint32_t g_vertexOffset = 0;
        const uint32_t g_numElementsToStore = 3 + 3 + 2; // pos(vec3) + normal(vec3) + uv(vec2)

        struct SceneConfig
        {
        	std::string fileName;
        	std::string outputMesh;
        	std::string outputScene;
        	std::string outputMaterials;
        	float scale;
        	bool calculateLODs;
        	bool mergeInstances;
        };
        
        void processScene(const SceneConfig& cfg)
        {
        	// clear mesh data from previous scene
        	g_MeshData.meshes_.clear();
        	g_MeshData.boxes_.clear();
        	g_MeshData.indexData_.clear();
        	g_MeshData.vertexData_.clear();

        	g_indexOffset = 0;
        	g_vertexOffset = 0;

        	// extract base model path
        	const std::size_t pathSeparator = cfg.fileName.find_last_of("/\\");
        	const std::string basePath = (pathSeparator != std::string::npos) ? cfg.fileName.substr(0, pathSeparator + 1) : std::string();

        	const unsigned int flags = 0 |
        		aiProcess_JoinIdenticalVertices |
        		aiProcess_Triangulate |
        		aiProcess_GenSmoothNormals |
        		aiProcess_LimitBoneWeights |
        		aiProcess_SplitLargeMeshes |
        		aiProcess_ImproveCacheLocality |
        		aiProcess_RemoveRedundantMaterials |
        		aiProcess_FindDegenerates |
        		aiProcess_FindInvalidData |
        		aiProcess_GenUVCoords;

        	printf("Loading scene from '%s'...\n", cfg.fileName.c_str());

        	const aiScene* scene = aiImportFile(cfg.fileName.c_str(), flags);

        	if (!scene || !scene->HasMeshes())
        	{
        		printf("Unable to load '%s'\n", cfg.fileName.c_str());
        		exit(EXIT_FAILURE);
        	}
            // 0. 以上是在加载模型，以及在模型加载前重置全局变量的状态

        	// 1. Mesh 数据转换
            // 1. Mesh conversion as in Chapter 5
        	g_MeshData.meshes_.reserve(scene->mNumMeshes);
        	g_MeshData.boxes_.reserve(scene->mNumMeshes);

        	for (unsigned int i = 0; i != scene->mNumMeshes; i++)
        	{
        		printf("\nConverting meshes %u/%u...", i + 1, scene->mNumMeshes);
        		Mesh mesh = convertAIMesh(scene->mMeshes[i], cfg);
        		g_MeshData.meshes_.push_back(mesh);
        	}

        	recalculateBoundingBoxes(g_MeshData);

        	saveMeshData(cfg.outputMesh.c_str(), g_MeshData);

        	Scene ourScene;

            // 2. 材质数据转换
        	// 2. Material conversion
        	std::vector<MaterialDescription> materials;
        	std::vector<std::string>& materialNames = ourScene.materialNames_;

        	std::vector<std::string> files;
        	std::vector<std::string> opacityMaps;

        	for (unsigned int m = 0 ; m < scene->mNumMaterials ; m++)
        	{
        		aiMaterial* mm = scene->mMaterials[m];

        		printf("Material [%s] %u\n", mm->GetName().C_Str(), m);
        		materialNames.push_back(std::string(mm->GetName().C_Str()));

        		MaterialDescription D = convertAIMaterialToDescription(mm, files, opacityMaps);
        		materials.push_back(D);
        		//dumpMaterial(files, D);
        	}

            // 3. 纹理图片处理
        	// 3. Texture processing, rescaling and packing
        	convertAndDownscaleAllTextures(materials, basePath, files, opacityMaps);

        	saveMaterials(cfg.outputMaterials.c_str(), materials, files);

            // 4. 场景数据转换
        	// 4. Scene hierarchy conversion
        	traverse(scene, ourScene, scene->mRootNode, -1, 0);

        	saveScene(cfg.outputScene.c_str(), ourScene);
        }

`processScene()` 函数主要分为三部分：
1. 转换 Mesh 几何数据，存储在全局变量中，并保存到文件中；
2. 转换 Material 材质数据，并处理纹理缩放等问题，并保存在文件中；
3. 转换场景图数据，和 Mesh、Material 产生关联，并保存在文件中；
下面一一分析：
1. Mesh 数据

        g_MeshData.meshes_.reserve(scene->mNumMeshes);
        g_MeshData.boxes_.reserve(scene->mNumMeshes);
        for (unsigned int i = 0; i != scene->mNumMeshes; i++)
        {
        	printf("\nConverting meshes %u/%u...", i + 1, scene->mNumMeshes);
        	Mesh mesh = convertAIMesh(scene->mMeshes[i], cfg);
        	g_MeshData.meshes_.push_back(mesh);
        }
        recalculateBoundingBoxes(g_MeshData);
        saveMeshData(cfg.outputMesh.c_str(), g_MeshData);
    其中，转换的关键在于 `convertAIMesh()` 函数

        Mesh convertAIMesh(const aiMesh* m, const SceneConfig& cfg)
        {
        	const bool hasTexCoords = m->HasTextureCoords(0);
        	const uint32_t streamElementSize = static_cast<uint32_t>(g_numElementsToStore * sizeof(float));

            // 1. 这里只用一个 stream，g_indexOffset 是之前所有 Mesh 的 indexCount 之和，g_vertexOffset 是之前所有 Mesh 的 vertexCount 之和
        	Mesh result = {
        		.streamCount = 1,
        		.indexOffset = g_indexOffset,
        		.vertexOffset = g_vertexOffset,
        		.vertexCount = m->mNumVertices,
        		.streamOffset = { g_vertexOffset * streamElementSize },
        		.streamElementSize = { streamElementSize }
        	};

        	// 2. 用来计算 LOD 的入参，以及保存计算 LOD 的结果
            // Original data for LOD calculation
        	std::vector<float> srcVertices;
        	std::vector<uint32_t> srcIndices;

        	std::vector<std::vector<uint32_t>> outLods;

            // 3. 开始保存顶点数据
        	auto& vertices = g_MeshData.vertexData_;

        	for (size_t i = 0; i != m->mNumVertices; i++)
        	{
        		// 3-1. 从模型中获取顶点、法线和纹理坐标
                const aiVector3D v = m->mVertices[i];
        		const aiVector3D n = m->mNormals[i];
        		const aiVector3D t = hasTexCoords ? m->mTextureCoords[0][i] : aiVector3D();

        		if (cfg.calculateLODs)
        		{
        			srcVertices.push_back(v.x);
        			srcVertices.push_back(v.y);
        			srcVertices.push_back(v.z);
        		}

                // 3-2. 保存顶点、法线和纹理坐标
        		vertices.push_back(v.x * cfg.scale);
        		vertices.push_back(v.y * cfg.scale);
        		vertices.push_back(v.z * cfg.scale);

        		vertices.push_back(t.x);
        		vertices.push_back(1.0f - t.y);

        		vertices.push_back(n.x);
        		vertices.push_back(n.y);
        		vertices.push_back(n.z);
        	}

            // 4. 开始保存索引数据
        	for (size_t i = 0; i != m->mNumFaces; i++)
        	{
        		if (m->mFaces[i].mNumIndices != 3)
        			continue;
        		for (unsigned j = 0; j != m->mFaces[i].mNumIndices; j++)
        			srcIndices.push_back(m->mFaces[i].mIndices[j]);
        	}

            // 5. 计算 LOD，这里数组的元素，都是一组对应 LOD 的索引数组
        	if (!cfg.calculateLODs)
        		outLods.push_back(srcIndices);
        	else
        		processLods(srcIndices, srcVertices, outLods);

        	printf("\nCalculated LOD count: %u\n", (unsigned)outLods.size());

            // 6. 保存经过 LOD 计算后的索引数据
        	uint32_t numIndices = 0;

        	for (size_t l = 0 ; l < outLods.size() ; l++)
        	{
        		for (size_t i = 0 ; i < outLods[l].size() ; i++)
        			g_MeshData.indexData_.push_back(outLods[l][i]);

        		result.lodOffset[l] = numIndices;
        		numIndices += (int)outLods[l].size();
        	}

        	result.lodOffset[outLods.size()] = numIndices;
        	result.lodCount = (uint32_t)outLods.size();

            // 7. 更新索引和顶点的 offset，是之前所有顶点和索引的数目之和
        	g_indexOffset  += numIndices;
        	g_vertexOffset += m->mNumVertices;

        	return result;
        }
    对于 LOD 计算的 `processLods` 函数，分析如下：

        void processLods(std::vector<uint32_t>& indices, std::vector<float>& vertices, std::vector<std::vector<uint32_t>>& outLods)
        {
        	// srcVertices 里面只保留了位置坐标，所以除以 3，可以计算得到顶点的数目
            size_t verticesCountIn = vertices.size() / 3;
        	size_t targetIndicesCount = indices.size();

        	uint8_t LOD = 1;

        	printf("\n   LOD0: %i indices", int(indices.size()));

            // 1. 一开始的索引数组，作为 LOD 为 0 的数据存储
        	outLods.push_back(indices);

            // 2. 只有在优化后索引数目小于 1024，或者 LOD >= 8 时，才会停止
        	while ( targetIndicesCount > 1024 && LOD < 8 )
        	{
        		// 3. 每次优化，计划比为上一级的一半
                targetIndicesCount = indices.size() / 2;

        		bool sloppy = false;

                // 4. 采用快速方法优化一波，得到最终的索引数据
        		size_t numOptIndices = meshopt_simplify(
        			indices.data(),
        			indices.data(), (uint32_t)indices.size(),
        			vertices.data(), verticesCountIn,
        			sizeof( float ) * 3,
        			targetIndicesCount, 0.02f );

                // 5. 如果压缩后的索引大小，乘以 1.1，比之前的索引还要多，说明快速压缩已经压不了多少了，那就换种方法，再压缩
        		// cannot simplify further
        		if (static_cast<size_t>(numOptIndices * 1.1f) > indices.size())
        		{
        			// 5-1. 如果 LOD == 1，说明是第一次压缩，结果都压的不理想，那干脆不压缩了
                    if (LOD > 1)
        			{
        				// try harder
        				numOptIndices = meshopt_simplifySloppy(
        					indices.data(),
        					indices.data(), indices.size(),
        					vertices.data(), verticesCountIn,
        					sizeof(float) * 3,
        					targetIndicesCount, 0.02f, nullptr);
        				sloppy = true;
                        // 5-2. 如果压缩不了了，干脆不压缩了
        				if (numOptIndices == indices.size()) break;
        			}
        			else
        				break;
        		}

                // 6. 重新调整索引数据大小
        		indices.resize(numOptIndices);
                
                // 7. 优化顶点数组
        		meshopt_optimizeVertexCache(indices.data(), indices.data(), indices.size(), verticesCountIn);

        		printf("\n   LOD%i: %i indices %s", int(LOD), int(numOptIndices), sloppy ? "[sloppy]" : "");

        		LOD++;
                
                // 8. 保存压缩结果
        		outLods.push_back(indices);
        	}
        }
    `convertAIMesh()` 函数计算完成后，转换后的数据保存在 g_MeshData 全局变量中，之后调用 `recalculateBoundingBoxes()` 函数，计算包围盒：
        // 包围盒的计算其实就是计算所有顶点数据中，最小的 x、y、z 坐标和最大的 x、y、z 坐标，这里的计算方法通过索引来找到顶点坐标，其实也可以直接遍历顶点数据
        void recalculateBoundingBoxes(MeshData& m)
        {
        	m.boxes_.clear();

        	for (const auto& mesh : m.meshes_)
        	{
        		const auto numIndices = mesh.getLODIndicesCount(0);

        		glm::vec3 vmin(std::numeric_limits<float>::max());
        		glm::vec3 vmax(std::numeric_limits<float>::lowest());

        		for (auto i = 0; i != numIndices; i++)
        		{
        			auto vtxOffset = m.indexData_[mesh.indexOffset + i] + mesh.vertexOffset;
        			const float* vf = &m.vertexData_[vtxOffset * kMaxStreams];
        			vmin = glm::min(vmin, vec3(vf[0], vf[1], vf[2]));
        			vmax = glm::max(vmax, vec3(vf[0], vf[1], vf[2]));
        		}

        		m.boxes_.emplace_back(vmin, vmax);
        	}
        }
    `recalculateBoundingBoxes()` 函数计算完包围盒后，开始将几何数据保存到文件中 `saveMeshData(cfg.outputMesh.c_str(), g_MeshData);`

        void saveMeshData(const char* fileName, const MeshData& m)
        {
        	FILE *f = fopen(fileName, "wb");

        	const MeshFileHeader header = {
        		.magicValue = 0x12345678, // 魔数，验证这是否是 Mesh 文件
        		.meshCount = (uint32_t)m.meshes_.size(), // mesh 的数量
        		.dataBlockStartOffset = (uint32_t )(sizeof(MeshFileHeader) + m.meshes_.size() * sizeof(Mesh)), // 数据段距离文件开始的 offset，以字节为单位
        		.indexDataSize = (uint32_t)(m.indexData_.size() * sizeof(uint32_t)), // 索引数据段的大小
        		.vertexDataSize = (uint32_t)(m.vertexData_.size() * sizeof(float)) // 顶点数据段的大小
        	};

        	fwrite(&header, 1, sizeof(header), f);
        	fwrite(m.meshes_.data(), sizeof(Mesh), header.meshCount, f);
        	fwrite(m.boxes_.data(), sizeof(BoundingBox), header.meshCount, f);
        	fwrite(m.indexData_.data(), 1, header.indexDataSize, f);
        	fwrite(m.vertexData_.data(), 1, header.vertexDataSize, f);

        	fclose(f);
        }
2. Material 数据
	材质数据的转换分为两步，第一步转换，第二不保存，我们先看第一步

		std::vector<MaterialDescription> materials;
		std::vector<std::string>& materialNames = ourScene.materialNames_;
		
		// 1. 保存纹理文件名
		std::vector<std::string> files;
		// 2. 保存 opacityMaps 纹理文件名
		std::vector<std::string> opacityMaps;

		for (unsigned int m = 0 ; m < scene->mNumMaterials ; m++)
		{
			aiMaterial* mm = scene->mMaterials[m];

			printf("Material [%s] %u\n", mm->GetName().C_Str(), m);
			materialNames.push_back(std::string(mm->GetName().C_Str()));

			MaterialDescription D = convertAIMaterialToDescription(mm, files, opacityMaps);
			materials.push_back(D);
			//dumpMaterial(files, D);
		}
	其中关键在 `convertAIMaterialToDescription()` 函数，其作用是将模型中的材质，转换为 `MaterialDescription` 的描述对象，下面详细分析下该函数

		MaterialDescription convertAIMaterialToDescription(const aiMaterial* M, std::vector<std::string>& files, std::vector<std::string>& opacityMaps)
		{
			MaterialDescription D;

			aiColor4D Color;

			// 1. 处理材质颜色
			if (aiGetMaterialColor(M, AI_MATKEY_COLOR_AMBIENT, &Color) == AI_SUCCESS)
			{
				D.emissiveColor_ = { Color.r, Color.g, Color.b, Color.a };
				if (D.emissiveColor_.w > 1.0f) D.emissiveColor_.w = 1.0f;
			}
			if (aiGetMaterialColor(M, AI_MATKEY_COLOR_DIFFUSE, &Color) == AI_SUCCESS)
			{
				D.albedoColor_ = { Color.r, Color.g, Color.b, Color.a };
				if (D.albedoColor_.w > 1.0f) D.albedoColor_.w = 1.0f;
			}
			if (aiGetMaterialColor(M, AI_MATKEY_COLOR_EMISSIVE, &Color) == AI_SUCCESS)
			{
				D.emissiveColor_.x += Color.r;
				D.emissiveColor_.y += Color.g;
				D.emissiveColor_.z += Color.b;
				D.emissiveColor_.w += Color.a;
				if (D.emissiveColor_.w > 1.0f) D.albedoColor_.w = 1.0f;
			}

			// 2. 处理透明
			const float opaquenessThreshold = 0.05f;
			float Opacity = 1.0f;

			if (aiGetMaterialFloat(M, AI_MATKEY_OPACITY, &Opacity) == AI_SUCCESS)
			{
				D.transparencyFactor_ = glm::clamp(1.0f - Opacity, 0.0f, 1.0f);
				if (D.transparencyFactor_ >= 1.0f - opaquenessThreshold) D.transparencyFactor_ = 0.0f;
			}

			if (aiGetMaterialColor(M, AI_MATKEY_COLOR_TRANSPARENT, &Color) == AI_SUCCESS)
			{
				const float Opacity = std::max(std::max(Color.r, Color.g), Color.b);
				D.transparencyFactor_ = glm::clamp(Opacity, 0.0f, 1.0f);
				if (D.transparencyFactor_ >= 1.0f - opaquenessThreshold) D.transparencyFactor_ = 0.0f;
				D.alphaTest_ = 0.5f;
			}

			float tmp = 1.0f;
			// 3. 金属度
			if (aiGetMaterialFloat(M, AI_MATKEY_GLTF_PBRMETALLICROUGHNESS_METALLIC_FACTOR, &tmp) == AI_SUCCESS)
				D.metallicFactor_ = tmp;

			// 4. 粗糙度
			if (aiGetMaterialFloat(M, AI_MATKEY_GLTF_PBRMETALLICROUGHNESS_ROUGHNESS_FACTOR, &tmp) == AI_SUCCESS)
				D.roughness_ = { tmp, tmp, tmp, tmp };

			aiString Path;
			aiTextureMapping Mapping;
			unsigned int UVIndex = 0;
			float Blend = 1.0f;
			aiTextureOp TextureOp = aiTextureOp_Add;
			aiTextureMapMode TextureMapMode[2] = { aiTextureMapMode_Wrap, aiTextureMapMode_Wrap };
			unsigned int TextureFlags = 0;

			// 5. emissiveMap 对应的纹理文件
			if ( aiGetMaterialTexture( M, aiTextureType_EMISSIVE, 0, &Path, &Mapping, &UVIndex, &Blend, &TextureOp, TextureMapMode, &TextureFlags ) == AI_SUCCESS )
			{
				D.emissiveMap_ = addUnique(files, Path.C_Str());
			}
			
			// 6. albedoMap 对应的纹理文件
			if ( aiGetMaterialTexture( M, aiTextureType_DIFFUSE, 0, &Path, &Mapping, &UVIndex, &Blend, &TextureOp, TextureMapMode, &TextureFlags ) == AI_SUCCESS )
			{
				D.albedoMap_ = addUnique(files, Path.C_Str());
				const std::string albedoMap = std::string(Path.C_Str());
				if (albedoMap.find("grey_30") != albedoMap.npos)
					D.flags_ |= sMaterialFlags_Transparent;
			}

			// 7. 切线空间法向量
			// first try tangent space normal map
			if ( aiGetMaterialTexture( M, aiTextureType_NORMALS, 0, &Path, &Mapping, &UVIndex, &Blend, &TextureOp, TextureMapMode, &TextureFlags) == AI_SUCCESS )
			{
				D.normalMap_ = addUnique(files, Path.C_Str());
			}
			// then height map
			if (D.normalMap_ == 0xFFFFFFFF)
				if ( aiGetMaterialTexture( M, aiTextureType_HEIGHT, 0, &Path, &Mapping, &UVIndex, &Blend, &TextureOp, TextureMapMode, &TextureFlags ) == AI_SUCCESS )
					D.normalMap_ = addUnique(files, Path.C_Str());
			
			// 8. opacityMap 透明图
			if ( aiGetMaterialTexture( M, aiTextureType_OPACITY, 0, &Path, &Mapping, &UVIndex, &Blend, &TextureOp, TextureMapMode, &TextureFlags ) == AI_SUCCESS )
			{
				D.opacityMap_ = addUnique(opacityMaps, Path.C_Str());
				D.alphaTest_ = 0.5f;
			}

			// 9. 根据素材名调整素材参数
			// patch materials
			aiString Name;
			std::string materialName;
			if (aiGetMaterialString(M, AI_MATKEY_NAME, &Name) == AI_SUCCESS)
			{
				materialName = Name.C_Str();
			}
			// apply heuristics
			if ((materialName.find("Glass") != std::string::npos) ||
				(materialName.find("Vespa_Headlight") != std::string::npos))
			{
				D.alphaTest_ = 0.75f;
				D.transparencyFactor_ = 0.1f;
				D.flags_ |= sMaterialFlags_Transparent;
			}
			else if (materialName.find("Bottle") != std::string::npos)
			{
				D.alphaTest_ = 0.54f;
				D.transparencyFactor_ = 0.4f;
				D.flags_ |= sMaterialFlags_Transparent;
			}
			else if (materialName.find("Metal") != std::string::npos)
			{
				D.metallicFactor_ = 1.0f;
				D.roughness_ = gpuvec4(0.1f, 0.1f, 0.0f, 0.0f);
			}

			return D;
		}
	之后，通过 `convertAndDownscaleAllTextures()` 函数进一步处理 albedo 纹理，把像素的 w 通道设置为 opacityMap 中的值；
	最后，通过 `saveMaterials()` 将处理后的 `MaterialDescription` 数组和纹理图像的位置数组，保存到文件中，如下：

		void saveMaterials(const char* fileName, const std::vector<MaterialDescription>& materials, const std::vector<std::string>& files)
		{
			FILE* f = fopen(fileName, "wb");
			if (!f)
				return;

			uint32_t sz = (uint32_t)materials.size();
			fwrite(&sz, 1, sizeof(uint32_t), f);
			fwrite(materials.data(), sizeof(MaterialDescription), sz, f);
			saveStringList(f, files);
			fclose(f);
		}

		void saveStringList(FILE* f, const std::vector<std::string>& lines)
		{
			uint32_t sz = (uint32_t)lines.size();
			fwrite(&sz, sizeof(uint32_t), 1, f);
			for (const auto& s: lines)
			{
				sz = (uint32_t)s.length();
				fwrite(&sz, sizeof(uint32_t), 1, f);
				fwrite(s.c_str(), sz + 1, 1, f);
			}
		}
	附上，加载材质的函数 `loadMaterials()` 的代码：

		void loadMaterials(const char* fileName, std::vector<MaterialDescription>& materials, std::vector<std::string>& files)
		{
			FILE* f = fopen(fileName, "rb");
			if (!f) {
				printf("Cannot load file %s\nPlease run SceneConverter tool from Chapter7\n", fileName);
				exit(255);
			}

			uint32_t sz;
			fread(&sz, 1, sizeof(uint32_t), f);
			materials.resize(sz);
			fread(materials.data(), sizeof(MaterialDescription), materials.size(), f);
			loadStringList(f, files);
			fclose(f);
		}

		void loadStringList(FILE* f, std::vector<std::string>& lines)
		{
			{
				uint32_t sz = 0;
				fread(&sz, sizeof(uint32_t), 1, f);
				lines.resize(sz);
			}
			std::vector<char> inBytes;
			for (auto& s: lines)
			{
				uint32_t sz = 0;
				fread(&sz, sizeof(uint32_t), 1, f);
				inBytes.resize(sz + 1);
				fread(inBytes.data(), sz + 1, 1, f);
				s = std::string(inBytes.data());
			}
		}
	
3. 场景图数据
首先，来看 `traverse()` 函数，主要功能是将模型中的节点保存为 Scene 中的一个节点，其中如果模型节点中含有 Mesh 几何数据，那么每个 Mesh 都添加为 Scene 中节点的子节点进行处理，然后遍历执行 `traverse()` 函数，处理模型节点的子节点

        void traverse(const aiScene* sourceScene, Scene& scene, aiNode* N, int parent, int ofs)
        {
        	// 1. 在 Scene 中添加子节点，返回值是 Scene 中该节点的数组索引，后面具体分析添加方法
            int newNode = addNode(scene, parent, ofs);

            // 2. 如果该节点有名字，则将名字保存在 names_ 数组中，并通过 nameForNode_ 这个 hashMap 进行查找
        	if (N->mName.C_Str())
        	{
        		makePrefix(ofs); printf("Node[%d].name = %s\n", newNode, N->mName.C_Str());

        		uint32_t stringID = (uint32_t)scene.names_.size();
        		scene.names_.push_back(std::string(N->mName.C_Str()));
        		scene.nameForNode_[newNode] = stringID;
        	}

            // 3. 如果该节点中有 Mesh 数据，每个 Mesh 都作为该节点的子节点出现
        	for (size_t i = 0; i < N->mNumMeshes ; i++)
        	{
        		// 3-1. 添加 Mesh 几何节点作为该节点的子节点
                int newSubNode = addNode(scene, newNode, ofs + 1);;
                
                // 3-2. 该 Mesh 命名为 xxx_Mesh_[i]
        		uint32_t stringID = (uint32_t)scene.names_.size();
        		scene.names_.push_back(std::string(N->mName.C_Str()) + "_Mesh_" + std::to_string(i));
        		scene.nameForNode_[newSubNode] = stringID;

                // 3-3. 保存该 Mesh 节点对应的几何数据和材质数据的索引，注意，这里利用了一个隐含条件，就是处理 Mesh 和 Material 材质时，是按照模型里面的顺序转换的，所以可以直接用模型里面的索引来获取处理后的数据
        		int mesh = (int)N->mMeshes[i];
        		scene.meshes_[newSubNode] = mesh;
        		scene.materialForNode_[newSubNode] = sourceScene->mMeshes[mesh]->mMaterialIndex;

                // 3-4. 打印调试信息
        		makePrefix(ofs); printf("Node[%d].SubNode[%d].mesh     = %d\n", newNode, newSubNode, (int)mesh);
        		makePrefix(ofs); printf("Node[%d].SubNode[%d].material = %d\n", newNode, newSubNode, sourceScene->mMeshes[mesh]->mMaterialIndex);

                // 3-5. 几何节点的转换矩阵，设置为单位矩阵
        		scene.globalTransform_[newSubNode] = glm::mat4(1.0f);
        		scene.localTransform_[newSubNode] = glm::mat4(1.0f);
        	}

            // 4. 设置该节点的转换矩阵
        	scene.globalTransform_[newNode] = glm::mat4(1.0f);
        	scene.localTransform_[newNode] = toMat4(N->mTransformation);

        	if (N->mParent != nullptr) {
        		makePrefix(ofs); printf("\tNode[%d].parent         = %s\n", newNode, N->mParent->mName.C_Str());
        		makePrefix(ofs); printf("\tNode[%d].localTransform = ", newNode); printMat4(N->mTransformation); printf("\n");
        	}

            // 5. 如果模型包含子节点，以该节点为父，继续遍历处理子节点
        	for (unsigned int n = 0 ; n  < N->mNumChildren ; n++)
        		traverse(sourceScene, scene, N->mChildren[n], newNode, ofs + 1);
        }
	`traverse()` 函数会深度遍历处理所有模型的节点，转换为场景图中的 Node，并将模型节点的 Mesh 数据，挂载为 Node 的几何子节点，下面我们看一下 `addNode()` 函数，是如何添加 Node 子节点的

		int addNode(Scene& scene, int parent, int level)
		{
			int node = (int)scene.hierarchy_.size();
			{
				// TODO: resize aux arrays (local/global etc.)
				scene.localTransform_.push_back(glm::mat4(1.0f));
				scene.globalTransform_.push_back(glm::mat4(1.0f));
			}
			
			// 1. 在最末位添加新的子节点，首先设置好父节点
			scene.hierarchy_.push_back({ .parent_ = parent, .lastSibling_ = -1 });

			// 2. 如果不是根节点的话，进入里面的流程
			if (parent > -1)
			{
				// 2-1. 找到父节点的第一个孩子节点
				// find first item (sibling)
				int s = scene.hierarchy_[parent].firstChild_;

				// 2-2. 如果不存在，说明该节点是父节点的第一个孩子，设置父节点的 firstChild_ 属性，以及自己的 lastSibling_ 属性
				if (s == -1)
				{
					scene.hierarchy_[parent].firstChild_ = node;
					scene.hierarchy_[node].lastSibling_ = node;
				} else
				{
					// 2-3. 如果找到了父节点的第一个孩子，那么根据其上面的 lastSibling_ 属性，找到父节点的最后的一个孩子
					int dest = scene.hierarchy_[s].lastSibling_;
					if (dest <= -1)
					{
						// 2-4. 如果没有 lastSibling_ 属性，就通过第一个孩子的 nextSibling_ 属性，找到他的兄弟节点，然后再通过兄弟节点的 nextSibling_ 属性一层层找下去，直到最后一个 nextSibling_ 为 -1 的节点，就是最后的孩子
						// no cached lastSibling, iterate nextSibling indices
						for (dest = s; scene.hierarchy_[dest].nextSibling_ != -1; dest = scene.hierarchy_[dest].nextSibling_);
					}
					// 2-5. 更新最后的孩子的 nextSibling_ 属性，和第一个孩子的 lastSibling_ 属性，第一个孩子有缓存最后一个孩子位置的作用
					scene.hierarchy_[dest].nextSibling_ = node;
					scene.hierarchy_[s].lastSibling_ = node;
				}
			}

			// 3. 设置新节点的深度层级、第一个孩子和兄弟属性
			scene.hierarchy_[node].level_ = level;
			scene.hierarchy_[node].nextSibling_ = -1;
			scene.hierarchy_[node].firstChild_  = -1;
			return node;
		}
	遍历处理完节点后，需要保存节点数据 `saveScene()`

		void saveScene(const char* fileName, const Scene& scene)
		{
			FILE* f = fopen(fileName, "wb");

			const uint32_t sz = (uint32_t)scene.hierarchy_.size();
			fwrite(&sz, sizeof(sz), 1, f);

			fwrite(scene.localTransform_.data(), sizeof(glm::mat4), sz, f);
			fwrite(scene.globalTransform_.data(), sizeof(glm::mat4), sz, f);
			fwrite(scene.hierarchy_.data(), sizeof(Hierarchy), sz, f);

			// Mesh for node [index to some list of buffers]
			saveMap(f, scene.materialForNode_);
			saveMap(f, scene.meshes_);

			if (!scene.names_.empty() && !scene.nameForNode_.empty())
			{
				saveMap(f, scene.nameForNode_);
				saveStringList(f, scene.names_);

				saveStringList(f, scene.materialNames_);
			}
			fclose(f);
		}

		void saveMap(FILE* f, const std::unordered_map<uint32_t, uint32_t>& map)
		{
			std::vector<uint32_t> ms;
			ms.reserve(map.size() * 2);
			for (const auto& m : map)
			{
				ms.push_back(m.first);
				ms.push_back(m.second);
			}
			const uint32_t sz = static_cast<uint32_t>(ms.size());
			fwrite(&sz, sizeof(sz), 1, f);
			fwrite(ms.data(), sizeof(int), ms.size(), f);
		}
	保存函数很简洁明了，基本结构如下：
	1. 节点数目，一个字节；
	2. localTransform_、globalTransform_、hierarchy_；
	3. 保存 Map，materialForNode_ 和 meshes_；  
	
	同样，载入函数，就是这个的逆过程

		void loadMap(FILE* f, std::unordered_map<uint32_t, uint32_t>& map)
		{
			std::vector<uint32_t> ms;

			uint32_t sz = 0;
			fread(&sz, 1, sizeof(sz), f);

			ms.resize(sz);
			fread(ms.data(), sizeof(int), sz, f);
			for (size_t i = 0; i < (sz / 2) ; i++)
				map[ms[i * 2 + 0]] = ms[i * 2 + 1];
		}

		void loadScene(const char* fileName, Scene& scene)
		{
			FILE* f = fopen(fileName, "rb");

			if (!f)
			{
				printf("Cannot open scene file '%s'. Please run SceneConverter from Chapter7 and/or MergeMeshes from Chapter 9", fileName);
				return;
			}

			uint32_t sz = 0;
			fread(&sz, sizeof(sz), 1, f);

			scene.hierarchy_.resize(sz);
			scene.globalTransform_.resize(sz);
			scene.localTransform_.resize(sz);
			// TODO: check > -1
			// TODO: recalculate changedAtThisLevel() - find max depth of a node [or save scene.maxLevel]
			fread(scene.localTransform_.data(), sizeof(glm::mat4), sz, f);
			fread(scene.globalTransform_.data(), sizeof(glm::mat4), sz, f);
			fread(scene.hierarchy_.data(), sizeof(Hierarchy), sz, f);

			// Mesh for node [index to some list of buffers]
			loadMap(f, scene.materialForNode_);
			loadMap(f, scene.meshes_);

			if (!feof(f))
			{
				loadMap(f, scene.nameForNode_);
				loadStringList(f, scene.names_);

				loadStringList(f, scene.materialNames_);
			}

			fclose(f);
		}

