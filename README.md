import React, { useState, useEffect, useRef } from "react";
import { v4 as uuidv4 } from "uuid";
import { useNavigate } from "react-router-dom";
import { getAuth, onAuthStateChanged } from "firebase/auth";
import Layout from "../components/Layout";
import { toast } from "react-toastify";
import {
  getStorage,
  ref,
  uploadBytesResumable,
  getDownloadURL,
} from "firebase/storage";
import { db } from "../firebase.config";
import { addDoc, collection, serverTimestamp } from "firebase/firestore";

const CreateListing = () => {
  const [formData, setFormData] = useState({
    type: "rent",
    name: "",
    bedrooms: 1,
    bathrooms: 1,
    parking: false,
    furnished: false,
    address: "",
    offer: false,
    regularPrice: 0,
    discountedPrice: 0,
    images: {},
  });

  const {
    type,
    name,
    bedrooms,
    bathrooms,
    parking,
    furnished,
    address,
    offer,
    regularPrice,
    discountedPrice,
    images,
  } = formData;

  const auth = getAuth();
  const navigate = useNavigate();
  const isMounted = useRef(true);

  useEffect(() => {
    if (isMounted) {
      onAuthStateChanged(auth, (user) => {
        setFormData({
          ...formData,
          useRef: user.uid,
        });
      });
    } else {
      navigate("/SignIn");
    }
    // eslint-disable-next-line
  }, []);

  //mutate func
  const onChangeHandler = (e) => {
    let boolean = null;
    if (e.target.value === "true") {
      boolean = true;
    }
    if (e.target.value === "false") {
      boolean = false;
    }

    //files
    if (e.target.files) {
      setFormData((prevState) => ({
        ...prevState,
        images: e.target.files,
      }));
    }

    //text/booleans/number
    if (!e.target.files) {
      setFormData((prevState) => ({
        ...prevState,
        [e.target.id]: boolean ?? e.target.value,
      }));
    }
  };

  //form submit
  const onSubmit = async (e) => {
    e.preventDefault();
    // console.log(formData);
    if (discountedPrice >= regularPrice) {
      toast.error("Discount Price should be less than Regular Price");
      return;
    }
    if (images > 6) {
      toast.error("Max 6 Images can be selected");
      return;
    }
    let location;
    location = address;

    //store images to firebase storage
    const storeImage = async (image) => {
      return new Promise((resolve, reject) => {
        const storage = getStorage();
        const fileName = `${auth.currentUser.uid}-${image.name}-${uuidv4()}`;
        const storageRef = ref(storage, "images/" + fileName);
        const uploadTask = uploadBytesResumable(storageRef, image);
        uploadTask.on(
          "state_changed",
          (snapshot) => {
            const progress =
              (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
            console.log("upload is" + progress + "% done");
            switch (snapshot.state) {
              case "paused":
                console.log("upload is paused");
                break;
              case "running":
                console.log("upload is runnning");
                break;
            }
          },
          (error) => {
            reject(error);
          },
          //success
          () => {
            getDownloadURL(uploadTask.snapshot.ref).then((downloadURL) => {
              resolve(downloadURL);
            });
          }
        );
      });
    };
    const imgUrls = await Promise.all(
      [...images].map((image) => storeImage(image))
    ).catch(() => {
      toast.error("Images not uploaded");
      return;
    });
    console.log(imgUrls);

    //save form data
    const formDataCopy = {
      ...formData,
      imgUrls,
      timestamp: serverTimestamp(),
    };
    formData.location = address;
    delete formDataCopy.images;
    !formDataCopy.offer && delete formDataCopy.discountedPrice;
    const docRef = await addDoc(collection(db, "listings"), formDataCopy);
    console.log(docRef);
    toast.success("Listing Created!");
    navigate(`/CategoryName/${formDataCopy.type}/${docRef.id}`);
  };

  return (
    <Layout>
      {/* sell rent button */}
      <form className="CreateListing-form" onSubmit={onSubmit}>
        <div className="form-check">
          <input
            className="form-check-input"
            type="radio"
            value="rent"
            onChange={onChangeHandler}
            defaultChecked
            name="type"
            id="type"
          />
          <label className="form-check-label" htmlFor="rent">
            Rent
          </label>
        </div>
        <div className="form-check">
          <input
            className="form-check-input"
            type="radio"
            name="type"
            value="sale"
            onChange={onChangeHandler}
            id="type"
          />
          <label className="form-check-label" htmlFor="sale">
            Sale
          </label>
        </div>

        {/* name */}
        <div className="name">
          <label htmlFor="name" className="form-label">
            Name
          </label>
          <input
            type="text"
            className="form-control"
            id="name"
            value={name}
            onChange={onChangeHandler}
            required
          />
        </div>

        {/* bedrooms */}
        <div className="bedroom">
          <label htmlFor="bedrooms" className="form-label">
            Bedrooms
          </label>
          <input
            type="number"
            className="form-control"
            id="bedrooms"
            value={bedrooms}
            onChange={onChangeHandler}
            required
          />
        </div>

        {/* bathrooms */}
        <div className="bathroom">
          <label htmlFor="bathrooms" className="form-label">
            Bathrooms
          </label>
          <input
            type="number"
            className="form-control"
            id="bathrooms"
            value={bathrooms}
            onChange={onChangeHandler}
            required
          />
        </div>

        {/* parking */}
        <div className=" parking ">
          <label htmlFor="parking" className="form-label">
            Parking :
          </label>

          <div className="form-check">
            <input
              className="form-check-input"
              type="radio"
              value={true}
              onChange={onChangeHandler}
              name="parking"
              id="parking"
            />
            <label className="form-check-label" htmlFor="yes">
              Yes
            </label>
          </div>
          <div className="form-check">
            <input
              className="form-check-input"
              type="radio"
              name="parking"
              value={false}
              defaultChecked
              onChange={onChangeHandler}
              id="parking"
            />
            <label className="form-check-label" htmlFor="no">
              No
            </label>
          </div>
        </div>

        {/* furnished */}
        <div className=" furnished ">
          <label htmlFor="furnished" className="form-label">
            Furnished :
          </label>

          <div className="form-check">
            <input
              className="form-check-input"
              type="radio"
              value={true}
              onChange={onChangeHandler}
              name="furnished"
              id="furnished"
            />
            <label className="form-check-label" htmlFor="yes">
              Yes
            </label>
          </div>
          <div className="form-check ">
            <input
              className="form-check-input"
              type="radio"
              name="furnished"
              value={false}
              defaultChecked
              onChange={onChangeHandler}
              id="furnished"
            />
            <label className="form-check-label" htmlFor="no">
              No
            </label>
          </div>
        </div>

        {/* address */}
        <div className="address">
          <label htmlFor="address">Address :</label>
          <textarea
            className="form-control"
            placeholder="Enter Your Address"
            id="address"
            value={address}
            onChange={onChangeHandler}
            required
          />
        </div>

        {/* offers  */}
        <div className=" offers ">
          <label htmlFor="offer" className="form-label">
            Offer :
          </label>

          <div className="form-check">
            <input
              className="form-check-input"
              type="radio"
              value={true}
              onChange={onChangeHandler}
              name="offer"
              id="offer"
            />
            <label className="form-check-label" htmlFor="yes">
              Yes
            </label>
          </div>
          <div className="form-check">
            <input
              className="form-check-input"
              type="radio"
              name="offer"
              value={false}
              defaultChecked
              onChange={onChangeHandler}
              id="offer"
            />
            <label className="form-check-label" htmlFor="no">
              No
            </label>
          </div>
        </div>

        {/* regular price */}
        <div className="regular-price">
          <label htmlFor="name" className="form-label">
            Regular Price :
          </label>

          <input
            type="number"
            className="form-control "
            id="regularPrice"
            name="regularPrice"
            value={regularPrice}
            onChange={onChangeHandler}
            required
          />
          {type === "rent" && <p className="Rent-para">Rs / Month</p>}
        </div>

        {/* offer */}
        {offer && (
          <div className="offer">
            <label htmlFor="discountedPrice" className="form-label">
              Discounted Price :
            </label>

            <input
              type="number"
              className="form-control"
              id="discountedPrice"
              name="discountedPrice"
              value={discountedPrice}
              onChange={onChangeHandler}
              required
            />
          </div>
        )}

        {/* files images etc */}
        <div className="files-images">
          <label htmlFor="formFile" className="form-label">
            Select images :
          </label>
          <input
            className="form-control"
            type="file"
            id="images"
            name="images"
            onChange={onChangeHandler}
            max="6"
            accept=".jpg,.png,.jpeg"
            multiple
            required
          />
        </div>

        {/* submit button */}
        <div className="submit">
          <input
            disabled={!name || !address || !regularPrice || !images}
            className="submit-button"
            type="submit"
            value="Create Listing"
          />
        </div>
      </form>
    </Layout>
  );
};

export default CreateListing;
