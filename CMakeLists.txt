cmake_minimum_required(VERSION 4.3)
project(rigidbodies)

set(CMAKE_CXX_STANDARD 20)

add_executable(rigidbodies main.cpp
        rigidbodies/PhysicsTypes.h
        rigidbodies/Constants.h
        src/Collider.cpp
        src/Collider.h
        src/Rigidbody.cpp
        src/Rigidbody.h
        src/MassProperties.cpp
        src/MassProperties.h
        src/BodyBuilder.cpp
        src/BodyBuilder.h
        src/World.cpp
        src/World.h
        src/Raycast.cpp
        src/Raycast.h
        src/collision/GJK.cpp
        src/collision/GJK.h
        src/collision/EPA.cpp
        src/collision/EPA.h
        src/collision/ManifoldBuilder.cpp
        src/collision/ManifoldBuilder.h
        src/dynamics/Integrator.cpp
        src/dynamics/Integrator.h
        src/dynamics/CollisionResolver.cpp
        src/dynamics/CollisionResolver.h
        src/dynamics/PositionCorrection.cpp
        src/dynamics/PositionCorrection.h)
