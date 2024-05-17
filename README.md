# Instructions for testing locally

- Install **PostgreSQL** on your machine:
    - For Windows: [Download this installer](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893)
    - For Linux: [Follow the steps here](https://www.postgresql.org/download/linux/)
- After you've installed PostgreSQL, open your terminal, and execute:
    - On Windows: ``psql -U postgres``
    - On Linux: ``sudo -u postgres psql``
    - IF prompted, *the default password is: "postgres"*
- Then, create a databse called **"ter_archivage_db"**, using the command ``CREATE DATABASE ter_archivage_db;``.
- To run the program successfully, add the following environment variables (either through your Jetbrains IDE or through
  a terminal):
    - ``POSTGRES_USER`` and set it to ``postgres``
    - ``POSTGRES_PASSWORD`` and set it to ``postgres``
    - ``POSTGRES_TABLES`` and set it to **the path** to the SQL
      file [(bin/database/database.sql)](/ter_backend/bin/database/database.sql). For *example*, mine is located
      at ``D:/Projects/ter_archivage/ter_backend/bin/database/database.sql``
    - ``FS_STORAGE_BASE_PATH`` and set it to the full local path to the directory you wish to store the files in.
    - ``ENCRYPTION_PRIVATE_KEY_FILE`` the path to the file which holds the private key used to decrypt passwords.
    - ``ENCRYPTION_PUBLIC_KEY_FILE`` the path to the file which holds the public key for the front-end.

# Webservices

### It is very recommended to send these requests in ``JSON`` format.

## User route ``/user``

> [!NOTE]
> This route is **unauthenticated**

- ``/user/create`` to create a User, this webservice takes:
    - ``login`` the username of the user (must be unique)
    - ``password`` a hashed version of the user password, hashed using **SHA256**
    - ``first_name``, ``last_name`` the user's name.

## Tags route ``/tags``

- ``/tags/create`` to create a Tag, this webservice takes:
    - ``tag_name`` the tag's name (must be unique)

    - ``/tags/all`` **GET**. Retrieves all the tags from the backend.
        - The response should look like:
      ```json5
      [
         {
          "id": 0,
          "tag_name": "Police"
         },
         {
          "id": 1,
          "tag_name": "Confidential"
         }     
      ]
        ```

## Documents route ``/document``

- ``/document/create`` to create a Document, this webservice takes:
    - ``type`` the document's type, optional unless the server is unable to recognize it.
        - ``type`` must be one of the following:
            - Text
            - PDF
            - Image
            - For the full implementation, look [here](/ter_backend/bin/database/elements/document.dart)
    - ``user_id`` the owner of the document. *(this field will be deprecated and removed as soon as authentication is
      added)*
    - ``path`` the remote path (directory) to store the document in.
        - This ``path`` must not start with neither ``.``  nor a ``/`` to prevent basic path-insertion attacks.
    - ``file`` the document file to upload to the server.
    - ``file_name`` the remote document file name (not final.)

  If the request is successful, the response of this request will be the created document's id in the database.


- ``/document/fetch`` fetches the documents made by a specified user. (will probably get removed in favor of filters.)
  it takes:
    - ``user`` the user to get the documents of.


- ``/document/update`` updates a document with new data.
    - ***The fields this webservice takes are optional, but at least one must be specified.***
    - ``new_path`` the new remote path to move the document to.
    - ``new_name`` the new file name for the document.
    - ``set_archived`` updates the document's archived status. *(when authentication is added, this field will check for
      the authenticated user's permission first.)*
    - ``new_type`` the new document type to set.

- ``/document/search`` searches through the documents through filters; this webservice takes:
    - ***The fields this webservice takes are optional, but at least one must be specified.***
    - ``filters`` the filters to use when searching.
        - This field is a list and each element must be structured in this manner:
            - ``name`` the filter's name
            - ... the filter's values.
        - The full list of filters and the values they take:
            - ``created_before``, ``created_after`` a filter to specify whether a document was **created** before/after
              a certain date, it takes:
            - ``date`` an integer representing the time in milliseconds using Epoch
            - ``modified_before``, ``modified_after`` a filter to specify whether a document was **modified**
              before/after a certain date, it takes:
            - ``date`` an integer representing the time in milliseconds using Epoch
            - ``owner`` used to filter documents made by a specific user, it takes:
                - ``value`` the user's id.
            - ``tagged_with`` filters documents with a set of tags.
                - ``tags`` a string list.
            - ``is_archived`` filters (un)archived documents, takes boolean ``value``
        - Example fully structured request using ``JSON``:
          ```json5
          { // Head object
            "filters": [ // The filters list
              { // Filter object
                "name": "is_archived",
                "value": true
              },
              {
                "name": "by_user",
                "value": 1
              },
              {
                "name": "tags",
                "tags": [
                    "Divorce",
                    "Police"
                ]
              },
              {
                "name": "directory",
                "path": "Directory1/Subdirectory1/Subdir2" // Filters the documents so only those on this path and lower are returned.
              }
            ]
          }
          ``` 

- ``/document/add/tag`` adds tags to a document.
    - ``tags`` the tags to add to the documents. **(a JSON list of integers) (tag ids)**

- ``/document/add/role`` adds roles to a document.
    - ``roles`` the roles to add to the documents. **(a JSON list of integers) (role ids)**

# Roles Route ``/roles``

- ``/roles/create`` creates a new role.

# Invites Route ``/invites``

Adding a person to collaborate with you on a document / directory is done through an **invitation system**, where **the
owner or someone with appropriate permissions** sends an invitation
This route has two paths:

- ``/invites/send`` sends an invitation to a user and it takes:
    - ``user_id`` the user to invite.
    - ``resource_id`` the resource to invite to.
    - ``role`` the role to give if the invitation is accepted.

- ``/invites/reply`` accept or deny an invitation.
    - ``answer`` a **true or false**
    - ``invite_id`` the invitation to accept or deny.

- ``/invites/fetch`` **(GET)** fetches the user's pending invites.

# Authentication Route ``/auth``

> [!NOTE]
> This route is **unauthenticated**

> [!WARNING]
> **To guarantee consistency** between runtimes, all of us should use the same public and private key pairs.\
> The private and public keys for this repository are included [here](/ter_backend/keys)\
> Unless the **private key** was compromised, there is **no need** to generate new key pairs.
>
> If you really need to generate a new key pair, you could use websites
> like: [this one](https://travistidwell.com/jsencrypt/demo/) or apps like: [PuTTYgen](https://www.puttygen.com/)

- ``/auth/key`` returns a **public RSA key**.
    - The front-end should use this route to **encrypt the password** and send it when attempting to authenticate

- ``/auth/attempt`` attempts to authenticate a user given some credentials, it takes:
    - ``username`` the username to authenticate with.
    - ``encrypted_password`` the password to use, this password must be encrypted using the **RSA public key** obtained
      from ``/auth/key``
    - If the credentials mismatch or do not exist, a code 400 is returned, otherwise a user token in Base64 is returned.

## How to use the obtained authentication token.

> [!CAUTION]
> A token generated by the back-end has **no set lifetime** *(meaning that it is valid forever)* unless either the
> user's password or username change.\
> Any malicious person with a valid token could send requests as another.\
> Since we are using **RSA**, a user may have an infinite amount of valid tokens to authenticate with.

- To authenticate a request, you must add a **Bearer token** in **the Authorization header** of your request.
- The headers should look something like:
    ```html
      Authorization: Bearer RrgZhGuks2nUsJy5RSSw5EX9Ba0R7QjgtVulLxzOaTRMFaOB5RNoRnhTyTkl5TV2jK249oBCfCKoHj7+KEpZ+Y9rLLPgA3kE9GCib8e42e9id+m+P3oTlHJPMmD3RjnV4Dh3ssuqULXFJLAZpLaqhw8Y1mgByFxSA37ue4aUQJKSt9lP9pfTTJ0I9vPb2lI5cRNG/bdVvco3bCpjRaqWqrcxcqXiuXkmEGsT2iSGvnUV+LjZIkudjIV1C7LH2ox2UASecJqcOPRdgxEVeECg0cnGfALUQHUnotZKXjWbvlpAAY9yPiksagiZ1i7q/DaFr0d9ltf0JD4RIfxgM+hrQQ==
      ... other headers like Content-Type etc...
   ```

# Front Route ``/front``

An authenticated route made to provide utility methods for the front-end.\
These utility methods include:

- ``/front/fetch/{user|role|tag|invite|resource|document}/{element_id}`` returns **the fields** the front-end might need
  to properly render a certain element.
    - If the authenticated sending user has permission to view **any** data on that element. It will be returned,
      otherwise, an **empty** JSON object will be returned instead.
    - Some elements might require **additional data** to properly return the view.
    - To supply said data, a body containing set fields must be sent, **depending** on the element.
        - ``resource`` element ***(only for directories)*** **optionally accepts**:
            - ``discover_subdirectories`` tells the back-end whether to fetch the contents of the subdirectories or not.
            - ``subdirectory_discovery_depth`` tells the back-end how many sub-subdirectories it should go when
              discovering.
