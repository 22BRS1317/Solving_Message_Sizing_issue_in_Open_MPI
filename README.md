#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <mpi.h>
#include <zlib.h>

#define MAX_MESSAGE_SIZE 1000

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    // Initialize message buffers
    char original_message[MAX_MESSAGE_SIZE];
    char compressed_message[MAX_MESSAGE_SIZE];
    char decompressed_message[MAX_MESSAGE_SIZE];

    // Generate a random message
    if (rank == 0) {
        for (int i = 0; i < MAX_MESSAGE_SIZE; i++) {
            original_message[i] = rand() % 26 + 'A'; // Random uppercase letters
        }
        original_message[MAX_MESSAGE_SIZE - 1] = '\0'; // Null-terminate the string
        printf("Original Message: %s\n", original_message);
    }

    // Compress the message on rank 0
    uLong compressed_size = sizeof(compressed_message);
    if (rank == 0) {
        int result = compress((Bytef*)compressed_message, &compressed_size, (const Bytef*)original_message, strlen(original_message) + 1);

        if (result == Z_OK) {
            // Successfully compressed the message
            // Send the compressed message size and the message to rank 1
            MPI_Send(&compressed_size, 1, MPI_UNSIGNED_LONG, 1, 0, MPI_COMM_WORLD);
            MPI_Send(compressed_message, compressed_size, MPI_CHAR, 1, 1, MPI_COMM_WORLD);
            printf("Compressed Message Sent to Rank 1\n");
        } else {
            fprintf(stderr, "Compression error on rank 0\n");
        }
    }

    // Receive and decompress the message on rank 1
    if (rank == 1) {
        MPI_Recv(&compressed_size, 1, MPI_UNSIGNED_LONG, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        MPI_Recv(compressed_message, compressed_size, MPI_CHAR, 0, 1, MPI_COMM_WORLD, MPI_STATUS_IGNORE);

        int result = uncompress((Bytef*)decompressed_message, (uLongf*)&compressed_size, (const Bytef*)compressed_message, compressed_size);

        if (result == Z_OK) {
            // Successfully decompressed the message
            printf("Decompressed Message on Rank 1: %s\n", decompressed_message);
        } else {
            fprintf(stderr, "Decompression error on rank 1\n");
        }
    }

    MPI_Finalize();
    return 0;
}
