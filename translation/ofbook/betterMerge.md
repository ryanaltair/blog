//改进 mergeDuplicateVertices
//----------------------------------------------------------
void ofMesh::mergeDuplicateVertices() {

	vector<ofVec3f> verts = getVertices();
	vector<ofIndexType> indices = getIndices();

	//get indexes to share single point - TODO: try j < i
    //
	for(ofIndexType i = 0; i < indices.size(); i++) {
		for(ofIndexType j = 0; j < indices.size(); j++ ) {
			if(i==j) continue;

			ofIndexType i1 = indices[i];
			ofIndexType i2 = indices[j];
			ofVec3f v1 = verts[ i1 ];
			ofVec3f v2 = verts[ i2 ];

			if( v1 == v2 && i1 != i2) {
				indices[j] = i1;
				break;
			}
		}
	}

	//indices array now has list of unique points we need
	//but we need to delete the old points we're not using and that means the index values will change
	//so we are going to create a new list of points and new indexes - we will use a map to map old index values to the new ones
	vector <ofPoint> newPoints;
	vector <ofIndexType> newIndexes;
	map <ofIndexType, bool> ptCreated;
	map <ofIndexType, ofIndexType> oldIndexNewIndex;

	vector<ofFloatColor> newColors;
	vector<ofFloatColor>& colors = getColors();
	vector<ofVec2f> newTCoords;
	vector<ofVec2f>& tcoords = getTexCoords();
	vector<ofVec3f> newNormals;
	vector<ofVec3f>& normals = getNormals();

	for(ofIndexType i = 0; i < indices.size(); i++){
		ptCreated[i] = false;
	}

	for(ofIndexType i = 0; i < indices.size(); i++){
		ofIndexType index = indices[i];
		ofPoint p = verts[ index ];

		if( ptCreated[index] == false ){
			oldIndexNewIndex[index] = newPoints.size();
			newPoints.push_back( p );
			if(hasColors()) {
				newColors.push_back(colors[index]);
			}
			if(hasTexCoords()) {
				newTCoords.push_back(tcoords[index]);
			}
			if(hasNormals()) {
				newNormals.push_back(normals[index]);
			}

			ptCreated[index] = true;
		}

		//ofLogNotice("ofMesh") << "[" << i << "]: old " << index << " --> " << oldIndexNewIndex[index];
		newIndexes.push_back( oldIndexNewIndex[index] );
	}

	verts.clear();
	verts = newPoints;

	indices.clear();
	indices = newIndexes;

	clearIndices();
	addIndices(indices);
	clearVertices();
	addVertices( verts );

	if(hasColors()) {
		clearColors();
		addColors( newColors );
	}

	if(hasTexCoords()) {
		clearTexCoords();
		addTexCoords( newTCoords );
	}

	if(hasNormals()) {
		clearNormals();
		addNormals( newNormals );
	}

}