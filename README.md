#!/bin/bash

# Predator_seq_removal
bash script to remove contamination from predator DNA in prey Sanger sequencing results

# Full predator sequence (this can be a longer sequence if needed)
predator_sequence="GCACAGCCCTTAGCCTGCTCATTCGGGCAGAACTAGCCCAACCCGGCGCCCTTCTAGGCGATGACCAAATTTATAACGTTATTGTTACTGCTCACGCCTTTGTAATAATTTTCTTTATAGTAATACCCATTATGATCGGAGGATTTGGAAATTGACTTGTGCCTCTAATAATTGGGGCCCCAGATATGGCCTTCCCCCGAATGAACAACATAAGCTTCTGACTCCTCCCTCCCTCCTTCCTACTTCTCCTCGCCTCCTCCGGGGTTGAGGCAGGAGCAGGAACAGGATGAACCGTGTACCCTCCCCTTGCCGGCAACCTCGCACACGCAGGGGCCTCCGTAGACTTAACCATCTTTTCCCTTCACCTCGCAGGAGTTTCATCTATTCTAGGAGCTATTAACTTTATTACAACCATCATTAATATAAAAAAACCCCCC..."

# Define the polymorphism codes manually as a series of if-else conditions
get_polymorphism_options() {
    case "$1" in
        R) echo "AG" ;;
        Y) echo "CT" ;;
        S) echo "GC" ;;
        W) echo "AT" ;;
        K) echo "GT" ;;
        M) echo "AC" ;;
        B) echo "CGT" ;;
        D) echo "AGT" ;;
        H) echo "ACT" ;;
        V) echo "ACG" ;;
        N) echo "ATCG" ;;
        *) echo "" ;;  # Return empty string if it's not a polymorphism
    esac
}

# Function to process the FASTA file
process_fasta() {
    input_fasta=$1
    output_fasta=$2

    # Variable to store the current prey sequence
    prey_sequence=""

    # Read the FASTA file line by line
    while IFS= read -r line; do
        if [[ $line == ">"* ]]; then
            # If it's a header line and we have a previous sequence, process it
            if [[ -n $prey_sequence ]]; then
                update_prey_sequence "$prey_sequence" >> "$output_fasta"
                prey_sequence=""  # Reset for the next sequence
            fi
            # Print the header line directly to the output
            echo "$line" >> "$output_fasta"
        else
            # If it's a sequence line, append it to the prey_sequence variable
            prey_sequence+="$line"
        fi
    done < "$input_fasta"

    # After the loop, check if there's a remaining prey sequence to process
    if [[ -n $prey_sequence ]]; then
        update_prey_sequence "$prey_sequence" >> "$output_fasta"
    fi
}

# Function to update prey sequence based on predator sequence
update_prey_sequence() {
    local prey_sequence="$1"

    # Convert predator and prey sequences to arrays of individual nucleotides
    predator_arr=($(echo "$predator_sequence" | grep -o .))
    prey_arr=($(echo "$prey_sequence" | grep -o .))

    # Initialize an array for the output sequence
    updated_prey_arr=()

    # Loop over the prey sequence and compare it with the predator sequence
    for (( i=0; i<${#prey_arr[@]}; i++ )); do
        prey_nt=${prey_arr[$i]}       # Prey nucleotide
        predator_nt=${predator_arr[$i]}   # Predator nucleotide

        # Get polymorphism options for the current nucleotide
        options=$(get_polymorphism_options "$prey_nt")

        if [[ -n "$options" ]]; then
            # Remove the predator's nucleotide from the options if it is not empty
            if [[ -n "$predator_nt" ]]; then
                remaining_nt=$(echo "$options" | sed "s/$predator_nt//g")
            else
                remaining_nt="$options"  # No predator nucleotide, keep all options
            fi

            # If there is a remaining nucleotide, replace it with the non-predator one
            if [[ -n "$remaining_nt" ]]; then
                updated_prey_arr+=("${remaining_nt:0:1}")
            else
                updated_prey_arr+=("$prey_nt")  # If no other options, keep the prey nucleotide
            fi
        else
            # If no polymorphism, just keep the original prey nucleotide
            updated_prey_arr+=("$prey_nt")
        fi
    done

    # Join the updated prey nucleotides into a single string
    updated_prey_sequence=$(printf "%s" "${updated_prey_arr[@]}")

    # Remove gaps (dashes) from the sequence (in case there are any)
    updated_prey_sequence=$(echo "$updated_prey_sequence" | tr -d '-')

    # Return the updated prey sequence
    echo "$updated_prey_sequence"
}

# Input and output files for analysis
input_fasta="00_raw_data/Sanger_results/Flathead_scDNA_Sanger.fas"   # Input FASTA file with prey sequences
output_fasta="01_Sanger_processing/Flathead_scDNA_Sanger_cleaned.fas" # Output FASTA file for updated prey sequences

# Clear the output file before appending to it
> "$output_fasta"

# Process the files for analysis
process_fasta "$input_fasta" "$output_fasta"

echo "Updated sequences have been written to $output_fasta"
