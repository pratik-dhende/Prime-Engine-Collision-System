# Prime Engine Collision System
## Demo Link: 
[https://drive.google.com/file/d/17MDN8JnqFatEunVUdM1APoA34Z8ScnDO/view?usp=drive_link](https://drive.google.com/file/d/17MDN8JnqFatEunVUdM1APoA34Z8ScnDO/view?usp=drive_link)

### Implementation details
Note: AABB and OBB will be used interchangebly in the following steps.
1. Created the following structs:
    1. `PhysicsManager`: A singleton for checking collisions.
    1. `PhysicsCollidable`: Represents a object which can collide with other objects
    1. `PhysicsCollidableSphere`: Sphere shape collidable object
    1. `PhysicsCollidableAABB`: An OBB collidable object.

1. Initialized the `PhysicsManager` inside `ClientGame::initEngine`.

1. Each `PhysicsCollidable` stores the world transform of itself and has virtual functions for collision checks between:
    1. `PhysicsCollidable` and `PhysicsCollidableSphere`
    1. `PhysicsCollidable` and `PhysicsCollidableAABB`
    1. `PhysicsCollidable` and `PhysicsCollidable`

1. Since, the `PhysicsCollidableAABB` and `PhysicsCollidableSphere` are derived from `PhysicsCollidable`, they override these functions for their respective collision checks.

1. For the case of `PhysicsCollidable` vs `PhysicsCollidable`, I have used double dispatch method to call the appropriate collision check function.

1. Inside `PhysicsManager`, I created a `vector<PhyiscsCollidable*>`.

1. Inside the `SoldierNPC::SoldierNPC`, I create the `PhysicsCollidableSphere` based on the soldier scene node's world transfrom and custom radius provided to it. This `PhysicsCollidableSphere` is pushed into the `vector<PhyiscsCollidable*>`.

1. Inside the `GameObjectManager::do_CREATE_MESH`, I create `PhysicsCollidableAABB` for the static meshes based on the local AABB bounds calculated in the `MeshManager::getAsset()` function and stored in the `Mesh` struct. Since these AABB points are in local space, I first transform them into world space by multiplying it with the world tranform of it's parent's scene node and store these world transfrom AABB bounds and it's world transform matrix in the `PhysicsCollidableAABB` struct. This `PhysicsCollidableAABB` is pushed into the `vector<PhyiscsCollidable*>`.

1. Collision detection between AABB (or OBB in our case) and sphere algorithm:
    1. Since the center of the sphere, and the minimum and maximum bounds of the OBB are stored in the world space, we need to transform them into the local space of the OBB.
    1. So the world matrix stored in the `PhysicsCollidableAABB` is used to transform these points to local space of OBB by multiplying them by the inverse of the `PhysicsCollidableAABB` world transform matrix.
    1. Once we have all of them in the local space of OBB, we now calculate the closest point of OBB to the center of the sphere in the following way:
        - Code snippet: 
        ```
        closestAABBPoint.m_x = max(localMinPoint.m_x, min(localSphereCenter.m_x, localMaxPoint.m_x));
        closestAABBPoint.m_y = max(localMinPoint.m_y, min(localSphereCenter.m_y, localMaxPoint.m_y));
        closestAABBPoint.m_z = max(localMinPoint.m_z, min(localSphereCenter.m_z, localMaxPoint.m_z));
        ```
    1. Then this closest point to OBB and sphere's center is converted to world space.
    1. If the distance between the closest point to OBB and sphere's center is less than or equal to the radius, then the sphere is colliding.

1. Created a struct called `AABBSphereCollisionPoints` which stores the closest point to AABB and the sphere's center and returns this as an output for the collision detection function.

1. Inside the `SoldierNPCMovementSM::do_UPDATE`, before updating the soldier's position, the `PhysicsManager::checkCollision` function is called which checks whether this new position collides the soldier's `PhysicsCollidableSphere` with other `PhysicsCollidable` objects.

1. If the soldier collides, then the `AABBSphereCollisionPoints` are returned.

1. Created a `AABBSphereCollisionSolver` static struct which takes the `AABBSphereCollisionPoints` and soldier's current velocity to return a struct `AABBSphereResolvedCollision` which contains the collision normal and the resulting velocity of the soldier.

1. AABB Sphere collision resolving algorithm:
    1. The closest point to the OBB and the sphere's center in the world space are taken and the collision normal to OBB plane is calculated by subracting the `closestAABBPoint` from the `sphereCenter`. Then this normal is normalized.
    1. To get the new velocity direction:
        1. The current velocity's dot product is done with the collision normal to get the current velocity's projection length on the collision normal.
        1. The collision normal is then scaled by this projection length and is then subtracted from the current velocity vector. The result is normalized and represents the new velocity direction.

1. Once we get the result of the collision detection points as `AABBSphereCollisionPoints` in the `SoldierNPCMovementSM::do_UPDATE`:
    1. If there is no collision, the `AABBSphereCollisionPoints` would be `nullptr` and the default behaviour would continue.
    1. If there is a collision, the `AABBSphereCollisionPoints` are passed onto the `AABBSphereCollisionSolver` to get the `AABBSphereResolvedCollision` which contains the collision normal and the resulting velocity direction.

1. The soldier is then translated along the collision normal by `PhysicsCollidableSphere.radius - distance(PhysicsCollidableSphere.center, closestOBBPoint)` and it's new velocity direction is set from the `AABBSphereResolvedCollision.velocityDir` and it's velocity magnitude is the same as it's last velocity magnitude.

1. Making the soldier fall through the cobble plane:

    1. Added a `bool isTrigger` member variable to `PhysicsCollidable` to not perform any collision resolving on it and instead just return whether a `PhysicsCollidable` triggers it or not.

    1. This `isTrigger` is set to true for cobble plane's `PhysicsCollidableAABB`.

    1. Collision detection is done between the soldier's `PhysicsCollidableSphere` and cobble plane's `PhysicsCollidableAABB`.

    1. If the soldier collides the plane then it stays on the cobble plane and if it doesn't collide with the cobble plane, the soldier is made to fall down.
    
    1. The soldier falls down by accelerating it using the formula `v = u + at` with `a = 9.8`.

    1. Inside the `SoldierNPCMovementSM::do_UPDATE`, these checks are done where the `PhysicsManager` is called to check whether the soldier's `PhysicsCollidableSphere` collides with the cobble plane's `PhysicsCollidableAABB` and triggers it or not.

1. Updated the level in Maya for appropriate demonstration of the task.
