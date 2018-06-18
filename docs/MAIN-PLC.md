```pascal
PROGRAM MAIN
VAR
    heartbeat       : UDINT;
    
    enc_raw AT %I*  : UINT;
    enc_angle       : REAL;
    
    // Time
    tDC             : T_DCTIME64;
    sTime           : STRING;
    dDate           : TIMESTRUCT;
    
    // File writing
    bRecordData     : BOOL;
    iFileState      : UINT;
    sFilePath       : STRING       := 'C:\Users\Public\Documents\EncoderData\';
    sFileType       : STRING       := '.bin';
    sFileName       : STRING;
    fbRecordTrigger : R_TRIG;
    
    fbFileOpen      : FB_FileOpen;              // Opens file
    fbFileClose     : FB_FileClose;             // Closes file
    fbFileWriter    : FB_FileWrite;             // Writes binary
    
    bBusy           : BOOL;
    bError          : BOOL;
    nErrId          : UDINT;
    hFile           : UINT         := 0;
    
    // Buffer
    stBuffer        : ARRAY [0..iBufferLength] OF DCTIME64_UINT;    // Buffer to dump data into
    iEndOfBuffer    : UDINT        := 0;                            // Index of the end of the buffer
    bBufferFull     : BOOL;                                         // Is stBuffer full?
    stBufferToWrite : ARRAY [0..iBufferLength] OF DCTIME64_UINT;    // Buffer to write into file (static copy of stBuffer)
    iLenToWrite     : UDINT;                                        // Number of elements in stBufferToWrite

END_VAR
VAR CONSTANT
    iBufferLength   : UDINT        := 50;                           // Number of elements in the array
    cbElementSize   : UDINT        := SIZEOF(DCTIME64_UINT);        // Size of each element
END_VAR
```

```pascal
heartbeat := heartbeat + 1;

// Get the encoder value
enc_angle := enc_raw MOD (360*4);
enc_angle := enc_angle / 4;


// Get the time
tDC := F_GetCurDcTaskTime64();
sTime := DCTIME64_TO_STRING(tDC);
dDate := DCTIME64_TO_SYSTEMTIME(tDC);


// Clear the buffer and set the file name when we start recording
fbRecordTrigger(CLK:=bRecordData);
IF fbRecordTrigger.Q THEN
    MEMSET( destAddr := ADR(stBuffer),
            fillByte := 0,
            n := cbElementSize * iBufferLength);
    iEndOfBuffer := 0;
    
    sFileName := GENERATE_FILENAME(sFilePath, dDate, sFileType);
END_IF


// If we're recording, and there's still room, add a new element to the buffer
IF bRecordData AND NOT bBufferFull THEN
    stBuffer[iEndOfBuffer].tDC := tDC;
    stBuffer[iEndOfBuffer].iData := enc_raw;
    
    iEndOfBuffer := iEndOfBuffer + 1;    
END_IF


// Check if the buffer is full
IF iBufferLength > iEndOfBuffer THEN
    bBufferFull := FALSE;
ELSE
    bBufferFull := TRUE;
END_IF


// State machine to handle writing the file
CASE iFileState OF
    0:    // Wait to start
        IF bRecordData THEN
            bBusy := TRUE;
            
            // Reset variables
            bError    := FALSE;
            nErrId    := 0;
            hFile     := 0;
            
            iFileState := 10;
        END_IF
        
    10:    // Open the file
        fbFileOpen(bExecute := FALSE);
        fbFileOpen( sNetId := '', 
                    sPathName := sFileName, 
                    nMode := FOPEN_MODEWRITE OR FOPEN_MODEBINARY, 
                    ePath := PATH_GENERIC, 
                    bExecute := TRUE);
        iFileState := 20;
    
    20:    // Wait for opening to finish
        fbFileOpen(bExecute := FALSE, bError => bError, nErrID => nErrID, hFile => hFile );
        IF NOT fbFileOpen.bBusy THEN
            IF NOT fbFileOpen.bError THEN
                iFileState := 30;
            ELSE
                // Error: file not found?
                iFileState := 100;
            END_IF
        END_IF
        
    30: // If we still want to write the file, write it
        IF bRecordData THEN
            // Clear stBufferToWrite                
            MEMSET( destAddr := ADR(stBufferToWrite),
                    fillByte := 0,
                    n := cbElementSize * iBufferLength);
            
            // Copy the stBuffer into stBufferToWrite
            MEMCPY( srcAddr := ADR(stBuffer),
                    destAddr := ADR(stBufferToWrite),
                    n := iEndOfBuffer * cbElementSize);
            
            // Write stBufferToWrite into the file
            fbFileWriter( sNetId := '',
                          hFile := hFile,
                          pWriteBuff := ADR(stBufferToWrite),
                          cbWriteLen := iEndOfBuffer * cbElementSize,
                          bExecute := TRUE);
            
            // Store the number of elements being written
            iLenToWrite := iEndOfBuffer;
            
            // Clear stBuffer                        
            MEMSET( destAddr := ADR(stBuffer),
                    fillByte := 0,
                    n := cbElementSize * iBufferLength);
            iEndOfBuffer := 0;
            
            iFileState := 40;
        
        ELSE // Stop writing
            iFileState := 200;
        END_IF
        
    40:    // Wait for the write to finish
        fbFileWriter( bExecute := FALSE, bError => bError, nErrID => nErrID );
        IF NOT fbFileWriter.bBusy THEN
            IF NOT fbFileWriter.bError THEN
                iFileState := 30;(* Write next record *)
            ELSE(* Error *)
                iFileState := 100;
            END_IF
        END_IF
            
    100: // Error
        IF ( hFile <> 0 ) THEN
            iFileState := 200; (* Close the source file *)
        ELSE
            bBusy := FALSE;
            iFileState := 0;    (* Ready *)
        END_IF
        
    200: // Close
        fbFileClose( bExecute := FALSE );
        fbFileClose( sNetId := '', hFile := hFile, bExecute := TRUE );
        iFileState := 210;
        
    210: // Wait for close to finish
        fbFileClose( bExecute := FALSE, bError => bError, nErrID => nErrID );
        IF ( NOT fbFileClose.bBusy ) THEN
            hFile := 0;
            bBusy := FALSE;
            iFileState := 0;    (* Ready *)
        END_IF

END_CASE
```