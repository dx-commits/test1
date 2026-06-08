# 1. Define your file paths
$groupFile  = "D:\Scripts\splitGroups\input_file.csv"   # The file with the comma-separated users
$mapFile    = "D:\Scripts\splitGroups\correctID.csv" # The file with ID and CorrectID
$outputFile = "D:\Scripts\splitGroups\final_output.csv"

# 2. Import files
$groupData = Import-Csv -Path $groupFile
$mapData   = Import-Csv -Path $mapFile

# 3. Create a fast lookup table (Hash Table) from the mapping file
# This maps the original ID to its CorrectID value
$idLookup = @{}
foreach ($row in $mapData) {
    if (-not [string]::IsNullOrWhiteSpace($row.ID)) {
        # Using .ToLower().Trim() ensures case-insensitive matching and strips trailing spaces
        $key = $row.ID.ToLower().Trim()
        $idLookup[$key] = $row.CorrectID.Trim()
    }
}

# 4. Process the group data and swap the IDs
$finalResults = @()

foreach ($row in $groupData) {
    $groupDomain1 = $row.GroupNameDomain1
    $groupDomain2 = $row.GroupNameDomain2
    
    # Split the comma-separated users in Column C
    $users = $row.Users -split ',' | ForEach-Object { $_.Trim() }
    
    foreach ($user in $users) {
        if (-not [string]::IsNullOrWhiteSpace($user)) {
            
            # Look up the corrected ID. If not found in the mapping file, default to the original user ID.
            $userKey = $user.ToLower().Trim()
            if ($idLookup.ContainsKey($userKey)) {
                $targetUser = $idLookup[$userKey]
            } else {
                $targetUser = $user # Fallback if no match is found
            }
            
            # Determine the domain from the updated user ID string
            $userDomain = ($targetUser -split '\\')[0]
            $matchedGroup = "No Domain Group Match Found"
            
            # Match the domain to the correct target group column
            if ($groupDomain1 -like "$userDomain\*") {
                $matchedGroup = $groupDomain1
            }
            elseif ($groupDomain2 -like "$userDomain\*") {
                $matchedGroup = $groupDomain2
            }
            
            # Append the clean row to our results collection
            $finalResults += [PSCustomObject]@{
                "User"        = $targetUser
                "TargetGroup" = $matchedGroup
            }
        }
    }
}

# 5. Export everything to your final clean CSV
$finalResults | Export-Csv -Path $outputFile -NoTypeInformation
Write-Host "Processing and ID mapping complete! Saved to: $outputFile" -ForegroundColor Green
