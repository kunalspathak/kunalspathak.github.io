
```cs
// Construction Phase

// Hierarchy
// RefPosition : Represents Use/Def/RegRecord

doLinearScan() {
    // Construction
    buildIntevals();    
    // Allocate
    allocateRegisters();
    // Resolution
    resolveRegisters();
}

#region Construction phase
// List of some important fields
class Interval {
    RefPosition firstRefPosition;
    RefPosition recentRefPosition;
    RefPosition lastRefPosition;

    bool isActive; // If this is currently in reg and alive
}

class RefPosition {
    RefPosition nextRefPosition;
}


// Build intervals for local variables and tree temps
// including the required RefPositions
buildIntervals() {
    // Build empty intervals for all physical registers
    buildPhysRegRecords();

    identifyCandidates();

    // iterate over method.Arguments to produce RefPositions
    for (block in method.Blocks) {
        for (node in block.GenTreeNodes) {
            buildRefPositionsForNode(node);
        }
    }
}

// Identify locals and compiler temps that are
// register candidates.
identifyCandidates() {
    for (lclNum in method.TotalLocals) {
        if (lclNum is register candidate) {
            newInterval = newInterval(); // create new interval 
            localVarIntervals[lclNum] = newInterval; // look-up table for lclVar : interval
        }
    }
}

// builds various RefPositions needed for this node
// as well as "tree temp" Intervals for a given node.
buildRefPositionsForNode(GenTree* node) {
    BuildNode(node);
}

// Builds the RegPosition for node `tree`
BuildNode(GenTree* tree) {
    switch (tree->OperGet()) {
        case GT_LCL_VAR:
            // Depending on node type, builds RefPositions as per this order:
            // 1. First, any internal registers are defined.
            //2. Next, any source registers are used (and are then freed if they are last use and are not identified as delayRegFree).
            //3. Next, the internal registers are used (and are then freed).
            //4. Next, any registers in the kill set for the instruction are killed.
            //5. Next, the destination register(s) are defined.
            //6. Finally, any delayRegFree source registers are freed.

            // Above operations are performed by Build*() methods
            break;
        // other cases
    }
}

// Build defintion RefPosition
BuildDef(GenTree* tree) {
    newInt = newInterval();
    defRefPosition = newRefPosition(interval); // RefTypeDef
    refInfo = listNodePool.GetNode(defRefPosition, tree); // adds to the listNodePool map
    defList.Append(refInfo); // definition list.
    return defRefPosition;
}

// Build use RefPostion
BuildUse(GenTree* operand) {
    if (localRef) {
        interval = localVarIntervals[operand->lclVarNum]
    } else {
        refInfo = defList.Remove(operand);
        defRefPosition = refInfo->ref;        
        interval = defRefPosition->interval;
    }
    useRefPos = newRefPosition(interval); // RefTypeUse
    return useRefPos;
}

// Creates new RefPosition for Interval/RegRecord
newRefPosition() {
    // checks which type of RefPosition to create
    newRefPos = newRefPositionRaw();
    newRefPos->setInterval(interval);

    // This method adds newRefPos to the interval
    // Interval mantains a list of refPos so it will basically
    // append to the list of existing refPos
    associateRefPosWithInterval(newRefPos);
}

newRefPositionRaw() {
    RefPosition* newRefPosition = // ... create new RefPosition*
    refPositions.Append(newRefPosition);
    return newRefPosition
}

#endregion

#region Allocation phase

allocateRegisters() {
    for(refPosition in refPositions) {
        // bunch of logic to spill/unassign registers, etc.
        // some continue to next refPositions

        assignedReg = tryAllocateFreeReg(currentInterval, currentRefPosition);
        if (assignedReg == REG_NA) {
            assignedReg = allocateBusyReg(currentInterval, currentRefPosition);
            // ...
        }

        currentRefPosition->registerAssignment = assignedReg;
        currentInterval->physReg = assignedReg;

    }

    freeRegisters(regsToFree | delayRegsToFree);
}

#endregion

#region Resolution phase
#endregion
```

TODO:
- How to read the JITDump for register allocator