



# File Structure

- Model
	- Rigged Model (Reference the Model)
	- Animation_X (Reference the Rigged Model)
	- Rigged Model Copy (Reference the Rigged Model)
		- REASON: since unity needs to have exact naming for each animation joint compared to the rigged model, we specifically need this because maya WILL export the "namespace:" name
		- This copy will do the same


# Creation

1. Create new scene
2. Reference to rigged model
3. Ensure X-Ray Joint is enabled in Viewport (to show them above mesh)
4. Create the Animation

# Export to Unity

Prereq:
- Export Joints and Mesh in "Rigged Model Copy" 

1. Key -> Bake Simulation
	1. Reason: this will bake the animation from controller to the joints itself
	2. NOTE: NVM, can just export controllers and joints, no need to bake
2. Selector controllers and joints
3. Export selection to unity